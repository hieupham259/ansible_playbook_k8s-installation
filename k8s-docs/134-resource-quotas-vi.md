# Hạn ngạch tài nguyên (Resource Quotas)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/resource-quotas/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7b](LO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên),
bài 3/6 · Kiểm chứng ở Lab 7b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là bài **dài nhất nhóm 7b** và hơn nửa số dòng là bảng tra tên tài nguyên cùng các mục con
về *Phạm vi hạn ngạch*. Đừng học thuộc bảng — chúng để tra khi viết YAML. Phần thực sự phải
nắm nằm trong hai mục đầu cộng mục *Hạn ngạch và dung lượng cluster*, khoảng 60 dòng. Giữ tiếp
câu phân biệt từ bài [133](133-limit-range-vi.md): LimitRange là trần cho **từng** đối tượng,
ResourceQuota là trần cho **tổng** cả namespace. Đây là công cụ chính khi chia cluster cho
nhiều nhóm.

**Phải hiểu ở lần đọc này:**

- ResourceQuota ràng buộc **tổng mức tiêu thụ cho mỗi namespace**, và chỉ được ép buộc trong
  namespace nào thực sự có một đối tượng ResourceQuota. Vi phạm thì control plane từ chối yêu
  cầu với `403 Forbidden` kèm thông báo giải thích ràng buộc lẽ ra sẽ bị vi phạm.
- Hệ quả bắt buộc phải nhớ: khi hạn ngạch áp cho `cpu` hoặc `memory`, **mọi Pod mới** phải chỉ
  định `requests` hoặc `limits` cho tài nguyên đó, nếu không control plane có thể từ chối
  admission. Với các tài nguyên khác thì hạn ngạch **bỏ qua** Pod không khai. Lối thoát mà bài
  chỉ ra: định nghĩa một [LimitRange](133-limit-range-vi.md) để ép giá trị mặc định.
- Bẫy Deployment: bạn thường không tạo Pod trực tiếp. Tạo một Deployment đòi nhiều hơn mức khả
  dụng vẫn **thành công**; chỉ là nó không đưa đủ số Pod vào tồn tại. Nơi tìm nguyên nhân là
  `kubectl describe` trên đối tượng quản lý workload, không phải ở thông báo của lệnh apply.
- Ba họ giới hạn mà ResourceQuota ép được: tài nguyên tính toán (`requests.cpu`,
  `limits.memory`…), lưu trữ (`requests.storage`, `persistentvolumeclaims`, tách theo
  StorageClass), và **số lượng đối tượng** (`count/<resource>.<group>`). Họ thứ ba tồn tại để
  bảo vệ bộ lưu trữ của control plane — ví dụ chặn CronJob hỏng đẻ ra vô số Job.
- ResourceQuota **độc lập với dung lượng cluster**: thêm node không tự cho namespace tiêu thụ
  nhiều hơn; ngược lại tổng hạn ngạch có thể vượt dung lượng thật và khi đó tranh chấp được xử
  lý theo nguyên tắc ai đến trước được phục vụ trước. Nó cũng **không tạo ràng buộc nào về
  node** — Pod của nhiều namespace vẫn chạy chung một node.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Hạn ngạch cho tài nguyên mở rộng* và *Hạn ngạch cho DRA resource claim* | chưa học DRA lẫn device plugin | giai đoạn 13, bài [149](149-dynamic-resource-allocation-vi.md); device plugin ở giai đoạn 14, bài [184](184-device-plugins-vi.md) |
| *Phạm vi hạn ngạch* và toàn bộ mục con (`BestEffort`, `Terminating`, `PriorityClass`, `CrossNamespacePodAffinity`, `VolumeAttributesClass`) | là tinh chỉnh nâng cao; hai ví dụ mạnh nhất còn cần cờ `--admission-control-config-file` của kube-apiserver | giai đoạn 9, bài [122](122-multi-tenancy-vi.md) khi ghép namespace + RBAC + quota thành mô hình cô lập |
| *Hạn ngạch cho lưu trữ tạm thời cục bộ* | vẫn ở mức alpha | giai đoạn 9, bài [122](122-multi-tenancy-vi.md) |
| Đoạn về custom resource và API aggregation trong *Hạn ngạch theo số lượng đối tượng* | chưa học CRD và aggregated API | giai đoạn 14, bài [179](179-custom-resources-vi.md) |
| Các bảng tra tên tài nguyên và output mẫu của `kubectl describe quota` | là tài liệu tra cứu, không phải nội dung để nhớ | tra lại khi viết YAML ở Lab 7b |

---

Khi nhiều người dùng hoặc nhiều nhóm cùng chia sẻ một cluster với số lượng node cố định,
sẽ có mối lo ngại rằng một nhóm nào đó có thể sử dụng nhiều hơn phần tài nguyên hợp lý của mình.

_Hạn ngạch tài nguyên (resource quota)_ là công cụ để quản trị viên giải quyết mối lo ngại này.

Một hạn ngạch tài nguyên, được định nghĩa bởi đối tượng ResourceQuota, cung cấp các ràng buộc giới hạn
tổng mức tiêu thụ tài nguyên cho mỗi namespace. Một ResourceQuota cũng có thể
giới hạn [số lượng đối tượng có thể được tạo trong một namespace](#quota-on-object-count) theo từng loại API (API kind), cũng như tổng
lượng tài nguyên hạ tầng (infrastructure resources) mà các đối tượng API trong namespace đó
có thể tiêu thụ.

> **Thận trọng:** Cả tranh chấp (contention) lẫn các thay đổi đối với hạn ngạch đều không ảnh hưởng đến các tài nguyên đã được tạo trước đó.

## Cách ResourceQuota trong Kubernetes hoạt động (How Kubernetes ResourceQuotas work)

ResourceQuota hoạt động như sau:

- Các nhóm khác nhau làm việc trong các namespace khác nhau. Sự phân tách này có thể được ép buộc bằng
  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) hoặc bất kỳ cơ chế [phân quyền (authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
  nào khác.

- Quản trị viên cluster tạo ít nhất một ResourceQuota cho mỗi namespace.
  - Để đảm bảo việc ép buộc luôn được duy trì, quản trị viên cluster cũng nên hạn chế quyền xóa hoặc cập nhật
    ResourceQuota đó; ví dụ, bằng cách định nghĩa một [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/).

- Người dùng tạo các tài nguyên (pod, service, v.v.) trong namespace, và hệ thống hạn ngạch
  theo dõi mức sử dụng để đảm bảo nó không vượt quá các giới hạn cứng (hard limit) được định nghĩa trong một ResourceQuota.

  Bạn có thể áp dụng một [phạm vi (scope)](#quota-scopes) cho một ResourceQuota để giới hạn nơi nó được áp dụng.

- Nếu việc tạo hoặc cập nhật một tài nguyên vi phạm một ràng buộc hạn ngạch, control plane sẽ từ chối yêu cầu đó với mã trạng thái HTTP
  `403 Forbidden`. Lỗi trả về bao gồm một thông báo giải thích ràng buộc lẽ ra sẽ bị vi phạm.

- Nếu hạn ngạch được bật trong một namespace cho các tài nguyên
  như `cpu` và `memory`, người dùng phải chỉ định request hoặc limit cho các giá trị đó khi định nghĩa một Pod; nếu không,
  hệ thống hạn ngạch có thể từ chối việc tạo pod.

  Bài [hướng dẫn thực hành](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/) về hạn ngạch tài nguyên
  cho thấy một ví dụ về cách tránh vấn đề này.

> **Ghi chú:**
> * Bạn có thể định nghĩa một [LimitRange](./133-limit-range-vi.md)
>   để ép buộc các giá trị mặc định cho các pod không đặt yêu cầu tài nguyên tính toán (để người dùng không phải nhớ tự làm việc đó).

Bạn thường không tạo Pod trực tiếp; ví dụ, bạn thường tạo một đối tượng [quản lý workload](./62-controllers-index-vi.md)
như Deployment. Nếu bạn tạo một Deployment cố gắng sử dụng nhiều
tài nguyên hơn mức khả dụng, việc tạo Deployment (hoặc đối tượng quản lý workload khác) **thành công**, nhưng
Deployment có thể không đưa được toàn bộ số Pod mà nó quản lý vào tồn tại. Trong trường hợp đó, bạn có thể kiểm tra trạng thái của
Deployment, ví dụ bằng `kubectl describe`, để xem điều gì đã xảy ra.

- Đối với tài nguyên `cpu` và `memory`, ResourceQuota ép buộc rằng **mọi**
  pod (mới) trong namespace đó phải đặt limit cho tài nguyên đó.
  Nếu bạn ép buộc một hạn ngạch tài nguyên trong một namespace cho `cpu` hoặc `memory`,
  bạn và các client khác **phải** chỉ định `requests` hoặc `limits` cho tài nguyên đó
  với mọi Pod mới mà bạn gửi lên. Nếu không, control plane có thể từ chối admission
  cho Pod đó.
- Đối với các tài nguyên khác: ResourceQuota vẫn hoạt động và sẽ bỏ qua các pod trong namespace không
  đặt limit hoặc request cho tài nguyên đó. Điều đó có nghĩa là bạn có thể tạo một pod mới
  không có limit/request cho lưu trữ tạm thời (ephemeral storage) ngay cả khi hạn ngạch tài nguyên giới hạn lưu trữ
  tạm thời của namespace này.

Bạn có thể sử dụng một [LimitRange](./133-limit-range-vi.md) để tự động đặt
request mặc định cho các tài nguyên này.

Tên của một đối tượng ResourceQuota phải là một
[tên miền con DNS hợp lệ](./17-names-vi.md#dns-subdomain-names).

Một số ví dụ về chính sách có thể được tạo bằng namespace và hạn ngạch là:

- Trong một cluster có dung lượng 32 GiB RAM và 16 core, cho nhóm A dùng 20 GiB và 10 core,
  cho nhóm B dùng 10 GiB và 4 core, và giữ lại 2 GiB cùng 2 core dự trữ cho việc cấp phát trong tương lai.
- Giới hạn namespace "testing" chỉ được dùng 1 core và 1 GiB RAM. Cho namespace "production"
  dùng không giới hạn.

Trong trường hợp tổng dung lượng của cluster nhỏ hơn tổng các hạn ngạch của các namespace,
có thể xảy ra tranh chấp tài nguyên. Việc này được xử lý theo nguyên tắc ai đến trước được phục vụ trước (first-come-first-served).

## Bật Resource Quota (Enabling Resource Quota)

Hỗ trợ ResourceQuota được bật mặc định trong nhiều bản phân phối Kubernetes. Nó được
bật khi cờ `--enable-admission-plugins=` của API server
có `ResourceQuota` là một trong các đối số của nó.

Một hạn ngạch tài nguyên được ép buộc trong một namespace cụ thể khi có một
ResourceQuota trong namespace đó.

## Các loại hạn ngạch tài nguyên (Types of resource quota)

Cơ chế ResourceQuota cho phép bạn ép buộc nhiều loại giới hạn khác nhau. Mục này
mô tả các loại giới hạn mà bạn có thể ép buộc.

### Hạn ngạch cho tài nguyên hạ tầng (Quota for infrastructure resources) {#compute-resource-quota}

Bạn có thể giới hạn tổng lượng
[tài nguyên tính toán](./110-manage-resources-containers-vi.md)
có thể được yêu cầu (request) trong một namespace nhất định.

Các loại tài nguyên sau được hỗ trợ:

| Tên tài nguyên | Mô tả |
| ------------- | ----------- |
| `limits.cpu` | Trên tất cả các pod ở trạng thái chưa kết thúc (non-terminal), tổng các limit CPU không được vượt quá giá trị này. |
| `limits.memory` | Trên tất cả các pod ở trạng thái chưa kết thúc, tổng các limit bộ nhớ không được vượt quá giá trị này. |
| `requests.cpu` | Trên tất cả các pod ở trạng thái chưa kết thúc, tổng các request CPU không được vượt quá giá trị này. |
| `requests.memory` | Trên tất cả các pod ở trạng thái chưa kết thúc, tổng các request bộ nhớ không được vượt quá giá trị này. |
| `hugepages-<size>` | Trên tất cả các pod ở trạng thái chưa kết thúc, số lượng request huge page với kích thước chỉ định không được vượt quá giá trị này. |
| `cpu` | Giống như `requests.cpu` |
| `memory` | Giống như `requests.memory` |

### Hạn ngạch cho tài nguyên mở rộng (Quota for extended resources)

Ngoài các tài nguyên nêu trên, ở phiên bản 1.10, hỗ trợ hạn ngạch cho
[tài nguyên mở rộng (extended resources)](./110-manage-resources-containers-vi.md#extended-resources) đã được bổ sung.

Vì overcommit không được phép đối với tài nguyên mở rộng, việc chỉ định cả `requests`
lẫn `limits` cho cùng một tài nguyên mở rộng trong một hạn ngạch là vô nghĩa. Vì vậy đối với tài nguyên mở rộng, chỉ các mục hạn ngạch
có tiền tố `requests.` được cho phép.

Lấy tài nguyên GPU làm ví dụ, nếu tên tài nguyên là `nvidia.com/gpu`, và bạn muốn
giới hạn tổng số GPU được yêu cầu trong một namespace là 4, bạn có thể định nghĩa hạn ngạch như sau:

* `requests.nvidia.com/gpu: 4`

Xem [Xem và thiết lập hạn ngạch](#viewing-and-setting-quotas) để biết thêm chi tiết.

### Hạn ngạch cho DRA resource claim (Quota for DRA resource claims)

Các resource claim của DRA (Dynamic Resource Allocation — Cấp phát tài nguyên động) có thể yêu cầu tài nguyên DRA theo device class. Với một device class ví dụ
tên là `examplegpu`, nếu bạn muốn giới hạn tổng số GPU được yêu cầu trong một namespace là 4,
bạn có thể định nghĩa hạn ngạch như sau:

* `examplegpu.deviceclass.resource.k8s.io/devices: 4`

Khi [cấp phát Extended Resource bằng DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource)
được bật, cùng device class tên `examplegpu` đó có thể được yêu cầu qua tài nguyên mở rộng, hoặc một cách tường minh
khi trường ExtendedResourceName của device class được chỉ định, chẳng hạn là `example.com/gpu`, khi đó bạn có thể định nghĩa hạn ngạch như sau:

* `requests.example.com/gpu: 4`

hoặc một cách ngầm định bằng tên tài nguyên mở rộng dẫn xuất từ tên device class `examplegpu`, bạn có thể định nghĩa
hạn ngạch như sau:

* `requests.deviceclass.resource.kubernetes.io/examplegpu: 4`

Tất cả các thiết bị được yêu cầu từ resource claim hoặc tài nguyên mở rộng đều được tính vào cả ba hạn ngạch
được liệt kê ở trên. Hạn ngạch tài nguyên mở rộng, ví dụ `requests.example.com/gpu: 4`, cũng tính các thiết bị được cung cấp
bởi device plugin.

Xem [Xem và thiết lập hạn ngạch](#viewing-and-setting-quotas) để biết thêm chi tiết.

### Hạn ngạch cho lưu trữ (Quota for storage)

Bạn có thể giới hạn tổng lượng [lưu trữ (storage)](./92-persistent-volumes-vi.md) cho các volume
có thể được yêu cầu trong một namespace nhất định.

Ngoài ra, bạn có thể giới hạn mức tiêu thụ tài nguyên lưu trữ dựa trên
[StorageClass](./96-storage-classes-vi.md) liên kết.

| Tên tài nguyên | Mô tả |
| ------------- | ----------- |
| `requests.storage` | Trên tất cả các persistent volume claim, tổng các request lưu trữ không được vượt quá giá trị này. |
| `persistentvolumeclaims` | Tổng số [PersistentVolumeClaim](./92-persistent-volumes-vi.md#persistentvolumeclaims) có thể tồn tại trong namespace. |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage` | Trên tất cả các persistent volume claim liên kết với `<storage-class-name>`, tổng các request lưu trữ không được vượt quá giá trị này. |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | Trên tất cả các persistent volume claim liên kết với `<storage-class-name>`, tổng số [persistent volume claim](./92-persistent-volumes-vi.md#persistentvolumeclaims) có thể tồn tại trong namespace. |

Ví dụ, nếu bạn muốn đặt hạn ngạch lưu trữ cho StorageClass `gold` tách biệt với
StorageClass `bronze`, bạn có thể định nghĩa hạn ngạch như sau:

* `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
* `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

#### Hạn ngạch cho lưu trữ tạm thời cục bộ (Quota for local ephemeral storage)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.8 [alpha]`

| Tên tài nguyên | Mô tả |
| ------------- | ----------- |
| `requests.ephemeral-storage` | Trên tất cả các pod trong namespace, tổng các request lưu trữ tạm thời cục bộ không được vượt quá giá trị này. |
| `limits.ephemeral-storage` | Trên tất cả các pod trong namespace, tổng các limit lưu trữ tạm thời cục bộ không được vượt quá giá trị này. |
| `ephemeral-storage` | Giống như `requests.ephemeral-storage`. |

> **Ghi chú:** Khi sử dụng một container runtime CRI, log của container sẽ được tính vào hạn ngạch lưu trữ tạm thời.
> Điều này có thể dẫn đến việc các pod đã dùng hết hạn ngạch lưu trữ của chúng bị trục xuất (evict) ngoài dự kiến.
>
> Tham khảo [Kiến trúc logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/) để biết chi tiết.

### Hạn ngạch theo số lượng đối tượng (Quota on object count) {#quota-on-object-count}

Bạn có thể đặt hạn ngạch cho *tổng số lượng của một loại tài nguyên cụ thể* trong API của Kubernetes,
bằng cú pháp sau:

* `count/<resource>.<group>` cho các tài nguyên thuộc các nhóm API không phải nhóm lõi (non-core API group)
* `count/<resource>` cho các tài nguyên thuộc nhóm API lõi (core API group)

Ví dụ, API PodTemplate nằm trong nhóm API lõi, do đó nếu bạn muốn giới hạn số lượng
đối tượng PodTemplate trong một namespace, bạn dùng `count/podtemplates`.

Những loại hạn ngạch này hữu ích để bảo vệ chống lại việc cạn kiệt bộ lưu trữ của control plane. Ví dụ, bạn có thể
muốn giới hạn số lượng Secret trong một server do kích thước lớn của chúng. Quá nhiều Secret trong một cluster
thực sự có thể khiến các server và controller không khởi động được. Bạn có thể đặt hạn ngạch cho Job để bảo vệ chống lại
một CronJob được cấu hình kém. Các CronJob tạo quá nhiều Job trong một namespace có thể dẫn đến từ chối dịch vụ (denial of service).

Nếu bạn định nghĩa hạn ngạch theo cách này, nó áp dụng cho các API của Kubernetes là một phần của API server, và
cho mọi custom resource được hỗ trợ bởi một CustomResourceDefinition.
Ví dụ, để tạo hạn ngạch cho custom resource `widgets` trong nhóm API `example.com`,
dùng `count/widgets.example.com`.
Nếu bạn dùng [API aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) để
thêm các API tùy chỉnh bổ sung không được định nghĩa dưới dạng CustomResourceDefinition, control plane
lõi của Kubernetes sẽ không ép buộc hạn ngạch cho API được tổng hợp (aggregated API) đó. Extension API server được kỳ vọng
sẽ tự cung cấp việc ép buộc hạn ngạch nếu điều đó phù hợp với API tùy chỉnh.

##### Cú pháp tổng quát (Generic syntax) {#resource-quota-object-count-generic}

Đây là danh sách các ví dụ phổ biến về những loại đối tượng bạn có thể muốn đưa vào hạn ngạch theo số lượng đối tượng,
được liệt kê theo chuỗi cấu hình mà bạn sẽ sử dụng.

* `count/pods`
* `count/persistentvolumeclaims`
* `count/services`
* `count/secrets`
* `count/configmaps`
* `count/deployments.apps`
* `count/replicasets.apps`
* `count/statefulsets.apps`
* `count/jobs.batch`
* `count/cronjobs.batch`

##### Cú pháp chuyên biệt (Specialized syntax) {#resource-quota-object-count-specialized}

Có một cú pháp khác cũng để đặt cùng loại hạn ngạch này, nhưng chỉ hoạt động với một số loại API nhất định.
Các loại sau được hỗ trợ:

| Tên tài nguyên | Mô tả |
| ------------- | ----------- |
| `configmaps` | Tổng số ConfigMap có thể tồn tại trong namespace. |
| `persistentvolumeclaims` | Tổng số [PersistentVolumeClaim](./92-persistent-volumes-vi.md#persistentvolumeclaims) có thể tồn tại trong namespace. |
| `pods` | Tổng số Pod ở trạng thái chưa kết thúc có thể tồn tại trong namespace. Một pod ở trạng thái kết thúc (terminal) nếu `.status.phase in (Failed, Succeeded)` là true. |
| `replicationcontrollers` | Tổng số ReplicationController có thể tồn tại trong namespace. |
| `resourcequotas` | Tổng số ResourceQuota có thể tồn tại trong namespace. |
| `services` | Tổng số Service có thể tồn tại trong namespace. |
| `services.loadbalancers` | Tổng số Service loại `LoadBalancer` có thể tồn tại trong namespace. |
| `services.nodeports` | Tổng số `NodePort` được cấp cho các Service loại `NodePort` hoặc `LoadBalancer` có thể tồn tại trong namespace. |
| `secrets` | Tổng số Secret có thể tồn tại trong namespace. |

Ví dụ, hạn ngạch `pods` đếm và ép buộc số lượng tối đa các `pods`
chưa kết thúc được tạo trong một namespace. Bạn có thể muốn đặt hạn ngạch `pods`
trên một namespace để tránh trường hợp một người dùng tạo nhiều pod nhỏ và
làm cạn kiệt nguồn cung địa chỉ IP dành cho Pod của cluster.

Bạn có thể xem thêm ví dụ tại [Xem và thiết lập hạn ngạch](#viewing-and-setting-quotas).

## Xem và thiết lập hạn ngạch (Viewing and Setting Quotas) {#viewing-and-setting-quotas}

kubectl hỗ trợ tạo, cập nhật và xem hạn ngạch:

```shell
kubectl create namespace myspace
```

```shell
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
    requests.nvidia.com/gpu: 4
EOF
```

```shell
kubectl create -f ./compute-resources.yaml --namespace=myspace
```

```shell
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    pods: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
```

```shell
kubectl create -f ./object-counts.yaml --namespace=myspace
```

```shell
kubectl get quota --namespace=myspace
```

```none
NAME                    AGE
compute-resources       30s
object-counts           32s
```

```shell
kubectl describe quota compute-resources --namespace=myspace
```

```none
Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
```

```shell
kubectl describe quota object-counts --namespace=myspace
```

```none
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
pods                    0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

kubectl cũng hỗ trợ hạn ngạch theo số lượng đối tượng cho tất cả các tài nguyên chuẩn thuộc phạm vi namespace
bằng cú pháp `count/<resource>.<group>`:

```shell
kubectl create namespace myspace
```

```shell
kubectl create quota test --hard=count/deployments.apps=2,count/replicasets.apps=4,count/pods=3,count/secrets=4 --namespace=myspace
```

```shell
kubectl create deployment nginx --image=nginx --namespace=myspace --replicas=2
```

```shell
kubectl describe quota --namespace=myspace
```

```
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.apps        1     2
count/pods                    2     3
count/replicasets.apps        1     4
count/secrets                 1     4
```

## Hạn ngạch và dung lượng cluster (Quota and Cluster Capacity)

Các ResourceQuota độc lập với dung lượng (capacity) của cluster. Chúng được
biểu diễn bằng đơn vị tuyệt đối. Vì vậy, nếu bạn thêm node vào cluster, điều này *không*
tự động cho phép mỗi namespace tiêu thụ nhiều tài nguyên hơn.

Đôi khi bạn có thể cần các chính sách phức tạp hơn, chẳng hạn:

- Chia tổng tài nguyên cluster theo tỷ lệ giữa nhiều nhóm.
- Cho phép mỗi bên thuê (tenant) tăng mức sử dụng tài nguyên khi cần, nhưng có một
  giới hạn rộng rãi để ngăn việc cạn kiệt tài nguyên do vô ý.
- Phát hiện nhu cầu từ một namespace, thêm node và tăng hạn ngạch.

Những chính sách như vậy có thể được triển khai bằng cách dùng `ResourceQuota` làm khối xây dựng (building block), bằng cách
viết một "controller" theo dõi mức sử dụng hạn ngạch và điều chỉnh các giới hạn cứng
của hạn ngạch cho mỗi namespace dựa trên các tín hiệu khác.

Lưu ý rằng hạn ngạch tài nguyên phân chia tổng tài nguyên của cluster, nhưng nó không tạo ra
ràng buộc nào liên quan đến node: các pod từ nhiều namespace có thể chạy trên cùng một node.

## Phạm vi hạn ngạch (Quota scopes) {#quota-scopes}

Mỗi hạn ngạch có thể có một tập các `scopes` (phạm vi) đi kèm. Một hạn ngạch sẽ chỉ đo mức sử dụng cho một tài nguyên nếu tài nguyên đó khớp
với giao (intersection) của các phạm vi được liệt kê.

Khi một phạm vi được thêm vào hạn ngạch, nó giới hạn số lượng loại tài nguyên mà hạn ngạch hỗ trợ xuống những loại liên quan đến phạm vi đó.
Các tài nguyên được chỉ định trong hạn ngạch nằm ngoài tập cho phép sẽ dẫn đến lỗi xác thực (validation error).

Kubernetes v1.36 hỗ trợ các phạm vi sau:

| Phạm vi | Mô tả |
| ----- | ----------- |
| [`BestEffort`](#quota-scope-best-effort) | Khớp các pod có chất lượng dịch vụ (quality of service) best effort. |
| [`CrossNamespacePodAffinity`](#cross-namespace-pod-affinity-scope) | Khớp các pod có các [điều khoản (anti)affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node) liên namespace (cross-namespace). |
| [`NotBestEffort`](#quota-scope-non-best-effort) | Khớp các pod không có chất lượng dịch vụ best effort. |
| [`NotTerminating`](#quota-scope-non-terminating) | Khớp các pod có `.spec.activeDeadlineSeconds` là `nil` |
| [`PriorityClass`](#resource-quota-per-priorityclass) | Khớp các pod tham chiếu đến [priority class](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption) được chỉ định. |
| [`Terminating`](#quota-scope-terminating) | Khớp các pod có `.spec.activeDeadlineSeconds` >= `0` |
| [`VolumeAttributesClass`](#quota-scope-volume-attributes-class) | Khớp các PersistentVolumeClaim tham chiếu đến [volume attributes class](./97-volume-attributes-classes-vi.md) được chỉ định. |

Các ResourceQuota có đặt phạm vi cũng có thể có trường `scopeSelector` tùy chọn. Bạn định nghĩa một hoặc nhiều _biểu thức khớp (match expression)_
chỉ định một `operator` và, nếu phù hợp, một tập `values` để so khớp. Ví dụ:

```yaml
  scopeSelector:
    matchExpressions:
      - scopeName: BestEffort # Khớp các pod có chất lượng dịch vụ best effort
        operator: Exists # tùy chọn; "Exists" được ngầm định cho phạm vi BestEffort
```

`scopeSelector` hỗ trợ các giá trị sau trong trường `operator`:

* `In`
* `NotIn`
* `Exists`
* `DoesNotExist`

Nếu `operator` là `In` hoặc `NotIn`, trường `values` phải có ít nhất
một giá trị. Ví dụ:

```yaml
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - middle
```

Nếu `operator` là `Exists` hoặc `DoesNotExist`, trường `values` *KHÔNG ĐƯỢC*
chỉ định.

### Phạm vi Pod best-effort (Best effort Pods scope) {#quota-scope-best-effort}

Phạm vi này chỉ theo dõi hạn ngạch tiêu thụ bởi các Pod.
Nó chỉ khớp các pod có [lớp QoS (QoS class)](./54-pod-qos-vi.md)
là [best effort](./54-pod-qos-vi.md#besteffort).

`operator` cho một `scopeSelector` phải là `Exists`.

### Phạm vi Pod không best-effort (Not-best-effort Pods scope) {#quota-scope-non-best-effort}

Phạm vi này chỉ theo dõi hạn ngạch tiêu thụ bởi các Pod.
Nó chỉ khớp các pod có [lớp QoS](./54-pod-qos-vi.md)
là [Guaranteed](./54-pod-qos-vi.md#guaranteed)
hoặc [Burstable](./54-pod-qos-vi.md#burstable).

`operator` cho một `scopeSelector` phải là `Exists`.

### Phạm vi Pod không kết thúc (Non-terminating Pods scope) {#quota-scope-non-terminating}

Phạm vi này chỉ theo dõi hạn ngạch tiêu thụ bởi các Pod không ở trạng thái kết thúc (not terminating). `operator` cho một `scopeSelector`
phải là `Exists`.

Một Pod không ở trạng thái kết thúc nếu trường `.spec.activeDeadlineSeconds` không được đặt.

Bạn có thể dùng một ResourceQuota với phạm vi này để quản lý các tài nguyên sau:

* `count.pods`
* `pods`
* `cpu`
* `memory`
* `requests.cpu`
* `requests.memory`
* `limits.cpu`
* `limits.memory`

### Phạm vi Pod đang kết thúc (Terminating Pods scope) {#quota-scope-terminating}

Phạm vi này chỉ theo dõi hạn ngạch tiêu thụ bởi các Pod đang kết thúc (terminating). `operator` cho một `scopeSelector`
phải là `Exists`.

Một Pod được coi là _đang kết thúc_ nếu trường `.spec.activeDeadlineSeconds` được đặt thành một số bất kỳ.

Bạn có thể dùng một ResourceQuota với phạm vi này để quản lý các tài nguyên sau:

* `count.pods`
* `pods`
* `cpu`
* `memory`
* `requests.cpu`
* `requests.memory`
* `limits.cpu`
* `limits.memory`

### Phạm vi pod affinity liên namespace (Cross-namespace pod affinity scope) {#cross-namespace-pod-affinity-scope}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Bạn có thể dùng [phạm vi hạn ngạch](#quota-scopes) `CrossNamespacePodAffinity` để giới hạn những namespace nào được phép
có các pod với điều khoản affinity vượt qua ranh giới namespace. Cụ thể, nó kiểm soát những pod nào được phép
đặt các trường `namespaces` hoặc `namespaceSelector` trong các [điều khoản (anti)affinity của pod](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node).

Việc ngăn người dùng sử dụng các điều khoản affinity liên namespace có thể là điều nên làm, vì một pod
với các ràng buộc anti-affinity có thể chặn các pod từ tất cả các namespace khác
không được xếp lịch vào một miền lỗi (failure domain).

Bằng phạm vi này, bạn (với vai trò quản trị viên cluster) có thể ngăn một số namespace nhất định — chẳng hạn `foo-ns` trong ví dụ dưới đây —
có các pod sử dụng pod affinity liên namespace. Bạn cấu hình điều này bằng cách tạo một đối tượng ResourceQuota trong
namespace đó với phạm vi `CrossNamespacePodAffinity` và giới hạn cứng là 0:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: disable-cross-namespace-affinity
  namespace: foo-ns
spec:
  hard:
    pods: "0"
  scopeSelector:
    matchExpressions:
    - scopeName: CrossNamespacePodAffinity
      operator: Exists
```

Nếu bạn muốn cấm sử dụng `namespaces` và `namespaceSelector` theo mặc định, và
chỉ cho phép chúng trong một số namespace cụ thể, bạn có thể cấu hình `CrossNamespacePodAffinity`
như một tài nguyên bị giới hạn (limited resource) bằng cách đặt cờ `--admission-control-config-file` của kube-apiserver
trỏ tới đường dẫn của file cấu hình sau:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: ResourceQuotaConfiguration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: CrossNamespacePodAffinity
        operator: Exists
```

Với cấu hình trên, các pod chỉ có thể dùng `namespaces` và `namespaceSelector` trong pod affinity
nếu namespace nơi chúng được tạo có một đối tượng resource quota với
phạm vi `CrossNamespacePodAffinity` và giới hạn cứng lớn hơn hoặc bằng số lượng pod đang sử dụng các trường đó.

### Phạm vi PriorityClass (PriorityClass scope) {#resource-quota-per-priorityclass}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.17 [stable]`

Một ResourceQuota với phạm vi PriorityClass chỉ khớp các Pod có một
[priority class](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption) cụ thể, và chỉ
khi có `scopeSelector` trong spec của hạn ngạch chọn đúng Pod đó.

Các Pod có thể được tạo với một [độ ưu tiên (priority)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority) cụ thể.
Bạn có thể kiểm soát mức tiêu thụ tài nguyên hệ thống của một pod dựa trên độ ưu tiên của pod đó, bằng cách dùng trường `scopeSelector`
trong spec của hạn ngạch.

Khi hạn ngạch được giới hạn phạm vi theo PriorityClass bằng trường `scopeSelector`, đối tượng ResourceQuota
chỉ có thể theo dõi (và giới hạn) các tài nguyên sau:

* `pods`
* `cpu`
* `memory`
* `ephemeral-storage`
* `limits.cpu`
* `limits.memory`
* `limits.ephemeral-storage`
* `requests.cpu`
* `requests.memory`
* `requests.ephemeral-storage`

#### Ví dụ (Example) {#quota-scope-priorityclass-example}

Ví dụ này tạo một ResourceQuota và khớp nó với các pod ở các độ ưu tiên cụ thể. Ví dụ
hoạt động như sau:

- Các pod trong cluster có một trong ba [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass): "low", "medium", "high".
  - Nếu bạn muốn thử ví dụ này, hãy dùng một cluster thử nghiệm và thiết lập ba PriorityClass đó trước khi tiếp tục.
- Một đối tượng hạn ngạch được tạo cho mỗi độ ưu tiên.

Hãy xem xét tập ResourceQuota sau:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-high
spec:
  hard:
    cpu: "1000"
    memory: "200Gi"
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-medium
spec:
  hard:
    cpu: "10"
    memory: "20Gi"
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["medium"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-low
spec:
  hard:
    cpu: "5"
    memory: "10Gi"
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["low"]
```

Áp dụng YAML bằng `kubectl create`.

```shell
kubectl create -f https://k8s.io/examples/policy/quota.yaml
```

```
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
```

Xác nhận rằng hạn ngạch `Used` là `0` bằng `kubectl describe quota`.

```shell
kubectl describe quota
```

```
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     1k
memory      0     200Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

Tạo một pod với độ ưu tiên "high".

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

Để tạo Pod:

```shell
kubectl create -f https://k8s.io/examples/policy/high-priority-pod.yaml

```

Xác nhận rằng thống kê "Used" của hạn ngạch cho độ ưu tiên "high", tức `pods-high`, đã thay đổi và
hai hạn ngạch còn lại không thay đổi.

```shell
kubectl describe quota
```

```
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         500m  1k
memory      10Gi  200Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

#### Giới hạn tiêu thụ PriorityClass theo mặc định (Limiting PriorityClass consumption by default)

Có thể bạn sẽ muốn rằng các pod ở một độ ưu tiên cụ thể, chẳng hạn "cluster-services",
chỉ được phép tồn tại trong một namespace khi và chỉ khi có một đối tượng hạn ngạch khớp tồn tại.

Với cơ chế này, người vận hành có thể hạn chế việc sử dụng một số
priority class cao chỉ trong một số lượng giới hạn các namespace, và không phải namespace nào
cũng có thể tiêu thụ các priority class này theo mặc định.

Để ép buộc điều này, cờ `--admission-control-config-file` của `kube-apiserver` nên được
dùng để truyền đường dẫn tới file cấu hình sau:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: ResourceQuotaConfiguration
    limitedResources:
    - resource: pods
      matchScopes:
      - scopeName: PriorityClass
        operator: In
        values: ["cluster-services"]
```

Sau đó, tạo một đối tượng resource quota trong namespace `kube-system`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-cluster-services
spec:
  scopeSelector:
    matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["cluster-services"]
```

```shell
kubectl apply -f https://k8s.io/examples/policy/priority-class-resourcequota.yaml -n kube-system
```

```none
resourcequota/pods-cluster-services created
```

Trong trường hợp này, việc tạo một pod sẽ được cho phép nếu:

1. `priorityClassName` của Pod không được chỉ định.
1. `priorityClassName` của Pod được chỉ định thành một giá trị khác `cluster-services`.
1. `priorityClassName` của Pod được đặt thành `cluster-services`, pod được tạo
   trong namespace `kube-system`, và nó đã vượt qua bước kiểm tra resource quota.

Một yêu cầu tạo Pod bị từ chối nếu `priorityClassName` của nó được đặt thành `cluster-services`
và nó được tạo trong một namespace khác `kube-system`.

### Phạm vi VolumeAttributesClass (VolumeAttributesClass scope) {#quota-scope-volume-attributes-class}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

Phạm vi này chỉ theo dõi hạn ngạch tiêu thụ bởi các PersistentVolumeClaim.

Các PersistentVolumeClaim có thể được tạo với một
[VolumeAttributesClass](./97-volume-attributes-classes-vi.md) cụ thể, và có thể được sửa đổi sau khi tạo.
Bạn có thể kiểm soát mức tiêu thụ tài nguyên lưu trữ của một PVC dựa trên các
VolumeAttributesClass liên kết, bằng cách dùng trường `scopeSelector` trong spec của hạn ngạch.

PVC tham chiếu VolumeAttributesClass liên kết qua các trường sau:

* `spec.volumeAttributesClassName`
* `status.currentVolumeAttributesClassName`
* `status.modifyVolumeStatus.targetVolumeAttributesClassName`

Một ResourceQuota liên quan chỉ được khớp và tiêu thụ khi ResourceQuota có một `scopeSelector` chọn đúng PVC đó.

Khi hạn ngạch được giới hạn phạm vi theo volume attributes class bằng trường `scopeSelector`, đối tượng hạn ngạch bị hạn chế chỉ theo dõi các tài nguyên sau:

* `persistentvolumeclaims`
* `requests.storage`

Đọc [Giới hạn tiêu thụ lưu trữ](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/) để tìm hiểu thêm về điều này.

## Tiếp theo (What's next)

- Xem một [ví dụ chi tiết về cách sử dụng resource quota](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/).
- Đọc [tài liệu tham chiếu API](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/) của ResourceQuota
- Tìm hiểu về [LimitRange](./133-limit-range-vi.md)
- Bạn có thể đọc [tài liệu thiết kế ResourceQuota](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_resource_quota.md)
  trong lịch sử để biết thêm thông tin.
- Bạn cũng có thể đọc [tài liệu thiết kế về hỗ trợ hạn ngạch cho priority class](https://git.k8s.io/design-proposals-archive/scheduling/pod-priority-resourcequota.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Namespace `team-a` có một ResourceQuota `requests.cpu: "2"` và một LimitRange
   `max.cpu: "1"`. Mỗi cái chặn điều gì? Nói rõ cái nào là trần cho từng đối tượng, cái nào là
   trần cho cả namespace, và một Pod xin `cpu: 1500m` sẽ bị cái nào từ chối.
2. Namespace đã có hạn ngạch cho `requests.memory`. Bạn `kubectl apply` một Deployment 5
   replica mà template không khai `requests` lẫn `limits` bộ nhớ. Lệnh apply báo gì, Pod ra
   sao, và bạn tìm nguyên nhân ở đâu?
3. Vẫn namespace đó, bạn không muốn người dùng phải nhớ khai `requests` nhưng cũng không muốn
   Pod bị từ chối. Theo bài, bạn thêm cái gì và nó can thiệp vào lúc nào?
4. Cluster lab của bạn có một control plane `k8s-master` và hai worker, mỗi worker 2 vCPU /
   6 GB RAM. Bạn thêm worker thứ ba. Hạn ngạch của `team-a` có tự nới ra không? Và
   ResourceQuota có buộc Pod của `team-a` chỉ chạy trên một số node nhất định không?
5. Hạn ngạch theo **số lượng đối tượng** bảo vệ được điều gì mà hạn ngạch CPU/memory không bảo
   vệ được? Nêu một ví dụ bài đưa ra.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **ResourceQuota là trần tổng của namespace; LimitRange là trần của từng đối tượng.**
   `requests.cpu: "2"` nghĩa là trên tất cả Pod chưa kết thúc trong `team-a`, **tổng** các
   request CPU không được vượt 2. `max.cpu: "1"` nghĩa là **mỗi** Container không được vượt 1.
   Pod xin `cpu: 1500m` bị **LimitRange** chặn — nó vi phạm trần đơn vị — kể cả khi namespace
   còn trống nguyên 2 CPU hạn ngạch. Ngược lại, ba Pod mỗi Pod `cpu: 1` đều lọt LimitRange
   nhưng Pod thứ ba sẽ bị **ResourceQuota** chặn vì tổng vượt 2. Hai cơ chế độc lập, cả hai
   cùng phải qua.
2. **Lệnh apply thành công, còn Pod thì không sinh ra đủ.** Bài nói rõ: bạn thường không tạo
   Pod trực tiếp, và nếu tạo một Deployment cố dùng nhiều tài nguyên hơn mức khả dụng thì việc
   tạo Deployment **thành công**, nhưng Deployment có thể không đưa được toàn bộ số Pod nó quản
   lý vào tồn tại. Lý do nền: khi hạn ngạch bật cho `memory`, mọi Pod mới **phải** chỉ định
   `requests` hoặc `limits` cho tài nguyên đó, nếu không control plane có thể từ chối admission
   — mà ở đây bên bị từ chối là các Pod do ReplicaSet tạo ra, không phải lệnh apply của bạn.
   Tìm nguyên nhân bằng **`kubectl describe` trên Deployment**, đúng như bài chỉ.
3. **Thêm một [LimitRange](133-limit-range-vi.md).** Bài ghi thẳng trong ghi chú: định nghĩa
   LimitRange để ép các giá trị mặc định cho những Pod không đặt yêu cầu tài nguyên tính toán,
   "để người dùng không phải nhớ tự làm việc đó". Nó can thiệp **ở giai đoạn admission, trước**
   khi hạn ngạch soi Pod: LimitRange chèn `requests` mặc định vào, nhờ đó Pod không còn rơi vào
   trường hợp thiếu request mà hạn ngạch từ chối. Đây là lý do hai đối tượng này gần như luôn đi
   thành cặp khi chia namespace cho nhiều nhóm.
4. **Không, và không.** Bài dành hẳn mục *Hạn ngạch và dung lượng cluster* cho ý đầu: các
   ResourceQuota **độc lập với dung lượng cluster** và được biểu diễn bằng đơn vị tuyệt đối, nên
   thêm node *không* tự động cho phép mỗi namespace tiêu thụ nhiều tài nguyên hơn — muốn nới thì
   phải sửa chính đối tượng hạn ngạch. Ý sau ở cuối cùng mục đó: hạn ngạch phân chia tổng tài
   nguyên của cluster nhưng **không tạo ràng buộc nào liên quan đến node** — Pod từ nhiều
   namespace vẫn chạy chung trên một node. Trên cluster của bạn, Pod của `team-a` và của
   namespace khác hoàn toàn có thể cùng nằm trên `k8s-worker1`.
5. **Bảo vệ bộ lưu trữ của control plane, thứ mà trần CPU/memory hoàn toàn không chạm tới.**
   Một đống đối tượng nhỏ xíu chẳng tốn CPU hay RAM của node nhưng vẫn phá được cluster. Ví dụ
   bài đưa ra: giới hạn số Secret vì kích thước lớn của chúng — quá nhiều Secret có thể khiến
   server và controller không khởi động được; và đặt hạn ngạch cho Job để chống một CronJob cấu
   hình kém tạo quá nhiều Job dẫn tới từ chối dịch vụ. Cú pháp là `count/<resource>` cho nhóm
   API lõi và `count/<resource>.<group>` cho nhóm khác.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
