# Cấp phát tài nguyên động (Dynamic Resource Allocation)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 1/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Đây là nhóm bài dành cho nền tảng chuyên biệt
(AI/HPC, GPU) và phần lớn nội dung còn ở trạng thái alpha hoặc beta. Chỉ đọc khi đã vững giai
đoạn 1–12, hoặc khi công việc thực sự cần. Cluster lab của bạn — 3 VM trên VMware, không GPU,
không thiết bị chuyên dụng — **không chạy được** phần lớn những gì bài này mô tả: không có
driver DRA thì cluster không có ResourceSlice nào để cấp phát. Mục tiêu của lần đọc này là
hiểu khái niệm, không phải thao tác.

Bản thân DRA đã `stable` từ v1.35, nhưng gần như mọi mục con trong bài vẫn là alpha/beta và
phụ thuộc feature gate riêng — **API của những phần đó có thể đổi giữa các phiên bản**. Bài
rất dài (hơn 1600 dòng); hơn một nửa độ dài nằm ở hai mục *Các tính năng beta của DRA* và
*Các tính năng alpha của DRA*, vốn viết cho tác giả driver chứ không phải quản trị viên.

**Phải hiểu ở lần đọc này:**

- Mô hình của DRA, qua phép so sánh ở mục *Giới thiệu về DRA*: DeviceClass đứng ở vị trí của
  StorageClass, ResourceClaim đứng ở vị trí của PersistentVolumeClaim. Pod tham chiếu claim,
  không tự khai số lượng thiết bị.
- Bốn kind API ở mục *Thuật ngữ DRA* và ai tạo cái nào: driver tạo **ResourceSlice** (công bố
  phần cứng), admin hoặc driver tạo **DeviceClass** (danh mục), người vận hành workload tạo
  **ResourceClaim** / **ResourceClaimTemplate**.
- Bốn lợi ích ở mục *Lợi ích của DRA* và đúng ba điểm bài nói device plugin không làm được:
  khai báo theo từng container, không chia sẻ thiết bị, không lọc theo biểu thức.
- Năm bước ở mục *Luồng công việc của Kubernetes*: tạo ResourceSlice → kiểm tra claim của
  workload → lọc ResourceSlice → cấp phát (first-fit) → lập lịch Pod lên node truy cập được
  thiết bị đã cấp phát.
- Mục *Hạn chế*: scheduler **không** hỗ trợ preemption cho tài nguyên DRA. Pod ưu tiên cao
  phải chờ Pod đang giữ thiết bị kết thúc hoặc bị xóa thủ công.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *ResourceClaim cho Workload* (`spec.resourceClaims` của PodGroup) | chưa học PodGroup và Workload | [bài 75](75-podgroup-api-vi.md) và [bài 77](77-workload-api-vi.md) cùng giai đoạn |
| *Ủy quyền trạng thái chi tiết*, các subresource tổng hợp | là phần hardening RBAC | [bài 125](125-hardening-dra-vi.md) cùng giai đoạn |
| *Danh sách ưu tiên*, *Đặt tên và mức ưu tiên* | là chi tiết tinh chỉnh cấp phát | khi công việc thực sự cần |
| *Khả năng quan sát tài nguyên động*, *Pod được lập lịch sẵn* | cần thiết bị thật để quan sát | khi công việc thực sự cần |
| Toàn bộ *Các tính năng beta của DRA* và *Các tính năng alpha của DRA* | viết cho tác giả driver, lab không có thiết bị | khi công việc thực sự cần |

Cách cũ để expose GPU và thiết bị — device plugin — nằm ở
[bài 184](184-device-plugins-vi.md) của giai đoạn 14. Nếu muốn so sánh hai cơ chế cạnh nhau,
đọc bài đó sau, đừng nhảy sang ngay.

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này mô tả cơ chế _cấp phát tài nguyên động (dynamic resource allocation — DRA)_ trong Kubernetes.

## Giới thiệu về DRA (About DRA) {#about-dra}

DRA là một tính năng của Kubernetes cho phép bạn yêu cầu (request) và chia sẻ tài nguyên
giữa các Pod. Các tài nguyên này thường là các thiết bị (device) gắn kèm,
chẳng hạn như bộ tăng tốc phần cứng (hardware accelerator).

Với DRA, các driver thiết bị và quản trị viên cluster định nghĩa các _lớp_ thiết bị
(device class) sẵn sàng để _claim_ (yêu cầu sử dụng) trong các workload. Kubernetes
cấp phát các thiết bị khớp với những claim cụ thể và đặt các Pod tương ứng
lên các node có thể truy cập những thiết bị đã được cấp phát.

Việc cấp phát tài nguyên bằng DRA mang lại trải nghiệm tương tự như
[cấp phát volume động (dynamic volume provisioning)](98-dynamic-provisioning-vi.md),
trong đó bạn dùng PersistentVolumeClaim để yêu cầu dung lượng lưu trữ từ các storage class
và sử dụng dung lượng đã claim đó trong các Pod của mình.

### Lợi ích của DRA (Benefits of DRA) {#dra-benefits}

DRA cung cấp một cách linh hoạt để phân loại, yêu cầu và sử dụng thiết bị trong cluster của bạn.
Sử dụng DRA mang lại những lợi ích như sau:

* **Lọc thiết bị linh hoạt**: dùng common expression language (CEL) để thực hiện
  lọc chi tiết theo các thuộc tính (attribute) cụ thể của thiết bị.
* **Chia sẻ thiết bị**: chia sẻ cùng một tài nguyên cho nhiều container hoặc nhiều Pod
  bằng cách tham chiếu đến resource claim tương ứng.
* **Phân loại thiết bị tập trung**: driver thiết bị và quản trị viên cluster có thể
  dùng các device class để cung cấp cho người vận hành ứng dụng những danh mục phần cứng
  đã được tối ưu cho nhiều trường hợp sử dụng khác nhau. Ví dụ, bạn có thể tạo một
  device class tối ưu chi phí cho các workload thông thường, và một device class
  hiệu năng cao cho các job quan trọng.
* **Đơn giản hóa yêu cầu của Pod**: với DRA, người vận hành ứng dụng không cần chỉ định
  số lượng thiết bị trong phần yêu cầu tài nguyên (resource request) của Pod. Thay vào đó,
  Pod tham chiếu đến một resource claim, và cấu hình thiết bị trong claim đó
  được áp dụng cho Pod.

Những lợi ích này mang lại các cải tiến đáng kể trong luồng cấp phát thiết bị
so với
[device plugin](184-device-plugins-vi.md),
vốn yêu cầu khai báo thiết bị theo từng container, không hỗ trợ chia sẻ thiết bị, và
không hỗ trợ lọc thiết bị dựa trên biểu thức.

### Các nhóm người dùng DRA (Types of DRA users) {#dra-user-types}

Luồng công việc sử dụng DRA để cấp phát thiết bị liên quan đến các nhóm người dùng sau:

* **Chủ sở hữu thiết bị (device owner)**: chịu trách nhiệm về thiết bị. Chủ sở hữu thiết bị
  có thể là nhà cung cấp thương mại, người vận hành cluster, hoặc một bên khác. Để dùng DRA,
  thiết bị phải có driver tương thích DRA thực hiện những việc sau:

  * Tạo các ResourceSlice cung cấp cho Kubernetes thông tin về
    node và tài nguyên.
  * Cập nhật ResourceSlice khi dung lượng tài nguyên trong cluster thay đổi.
  * Tùy chọn, tạo các DeviceClass mà người vận hành workload có thể dùng để
    claim thiết bị.

* **Quản trị viên cluster (cluster admin)**: chịu trách nhiệm cấu hình cluster và node,
  gắn thiết bị, cài đặt driver, và các công việc tương tự. Để dùng DRA,
  quản trị viên cluster thực hiện những việc sau:

  * Gắn thiết bị vào các node.
  * Cài đặt các driver thiết bị hỗ trợ DRA.
  * Tùy chọn, tạo các DeviceClass mà người vận hành workload có thể dùng để claim thiết bị.

* **Người vận hành workload (workload operator)**: chịu trách nhiệm triển khai và quản lý
  các workload trong cluster. Để dùng DRA cấp phát thiết bị cho các Pod, người vận hành
  workload thực hiện những việc sau:

  * Tạo các ResourceClaim hoặc ResourceClaimTemplate để yêu cầu các cấu hình
    cụ thể trong phạm vi các DeviceClass.
  * Triển khai các workload sử dụng những ResourceClaim hoặc ResourceClaimTemplate cụ thể.

## Thuật ngữ DRA (DRA terminology) {#terminology}

DRA dùng các loại (kind) API Kubernetes sau để cung cấp chức năng cấp phát
cốt lõi. Tất cả các kind API này đều thuộc nhóm API (API group) `resource.k8s.io/v1`.

DeviceClass
: Định nghĩa một danh mục thiết bị có thể được claim và cách chọn các thuộc tính
  thiết bị cụ thể trong claim. Các tham số của DeviceClass có thể khớp với không hoặc
  nhiều thiết bị trong các ResourceSlice. Để claim thiết bị từ một DeviceClass,
  các ResourceClaim chọn những thuộc tính thiết bị cụ thể.

ResourceClaim
: Mô tả một yêu cầu truy cập đến các tài nguyên gắn kèm trong cluster, chẳng hạn
  thiết bị. ResourceClaim cung cấp cho Pod quyền truy cập vào
  một tài nguyên cụ thể. ResourceClaim có thể do người vận hành workload tạo
  hoặc do Kubernetes sinh ra dựa trên một ResourceClaimTemplate.

ResourceClaimTemplate
: Định nghĩa một template mà Kubernetes dùng để tạo ResourceClaim theo từng Pod
  cho một workload. ResourceClaimTemplate cung cấp cho các Pod quyền truy cập vào
  những tài nguyên riêng biệt nhưng tương tự nhau. Mỗi ResourceClaim mà Kubernetes
  sinh ra từ template được gắn với một Pod cụ thể. Khi Pod
  kết thúc, Kubernetes xóa ResourceClaim tương ứng.

ResourceSlice
: Đại diện cho một hoặc nhiều tài nguyên được gắn vào các node, chẳng hạn thiết bị.
  Driver tạo và quản lý các ResourceSlice trong cluster. Khi một ResourceClaim
  được tạo và được dùng trong một Pod, Kubernetes dùng các ResourceSlice để tìm những node
  có quyền truy cập vào các tài nguyên được claim. Kubernetes cấp phát tài nguyên cho
  ResourceClaim và lập lịch Pod lên một node có thể truy cập các tài nguyên đó.

### DeviceClass {#deviceclass}

DeviceClass cho phép quản trị viên cluster hoặc driver thiết bị định nghĩa các danh mục thiết bị
trong cluster. DeviceClass cho người vận hành biết họ có thể yêu cầu những thiết bị nào và
cách yêu cầu những thiết bị đó. Bạn có thể dùng
[common expression language (CEL)](https://cel.dev) để chọn thiết bị dựa trên
các thuộc tính cụ thể. Một ResourceClaim tham chiếu đến DeviceClass sau đó có thể
yêu cầu các cấu hình cụ thể trong phạm vi DeviceClass đó.

Để tạo một DeviceClass, xem
[Thiết lập DRA trong cluster](271-set-up-dra-cluster-vi.md).

### ResourceClaim và ResourceClaimTemplate (ResourceClaims and ResourceClaimTemplates) {#resourceclaims-templates}

ResourceClaim định nghĩa các tài nguyên mà một workload cần. Mỗi ResourceClaim
có các _request_ tham chiếu đến một DeviceClass và chọn thiết bị từ
DeviceClass đó. ResourceClaim cũng có thể dùng _selector_ để lọc những thiết bị
đáp ứng các yêu cầu cụ thể, và có thể dùng _constraint_ (ràng buộc) để giới hạn những thiết bị
có thể thỏa mãn một request. ResourceClaim có thể do người vận hành workload tạo hoặc
có thể được Kubernetes sinh ra dựa trên một ResourceClaimTemplate.
ResourceClaimTemplate định nghĩa một template mà Kubernetes có thể dùng để
tự động sinh ResourceClaim cho các Pod.

#### Trường hợp sử dụng ResourceClaim và ResourceClaimTemplate (Use cases for ResourceClaims and ResourceClaimTemplates) {#when-to-use-rc-rct}

Phương thức bạn dùng phụ thuộc vào yêu cầu của bạn, như sau:

* **ResourceClaim**: bạn muốn nhiều Pod chia sẻ quyền truy cập vào những thiết bị
  cụ thể. Bạn tự quản lý vòng đời của các ResourceClaim mà bạn tạo.
* **ResourceClaimTemplate**: bạn muốn các Pod có quyền truy cập độc lập vào
  những thiết bị riêng biệt được cấu hình tương tự nhau. Kubernetes sinh các ResourceClaim
  từ đặc tả (specification) trong ResourceClaimTemplate. Thời gian tồn tại của mỗi
  ResourceClaim được sinh ra gắn với thời gian tồn tại của Pod tương ứng.
* [**ResourceClaimTemplate cho PodGroup**](#workload-resourceclaims): bạn muốn
  các PodGroup có quyền truy cập độc lập vào những thiết bị riêng biệt được cấu hình
  tương tự nhau, và các Pod trong nhóm có thể chia sẻ các thiết bị đó. Kubernetes
  sinh một ResourceClaim cho PodGroup từ đặc tả trong ResourceClaimTemplate.
  Thời gian tồn tại của mỗi ResourceClaim được sinh ra gắn với thời gian tồn tại
  của PodGroup tương ứng. Cách này yêu cầu bật tính năng
  [`DRAWorkloadResourceClaims`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAWorkloadResourceClaims).

Khi định nghĩa một workload, bạn có thể dùng
Common Expression Language (CEL)
để lọc theo các thuộc tính hoặc dung lượng (capacity) cụ thể của thiết bị. Các tham số
có thể dùng để lọc phụ thuộc vào thiết bị và driver.

Nếu bạn tham chiếu trực tiếp một ResourceClaim cụ thể trong Pod, ResourceClaim đó
phải đã tồn tại trong cùng namespace với Pod. Nếu ResourceClaim
không tồn tại trong namespace, Pod sẽ không được lập lịch. Hành vi này tương tự
việc một PersistentVolumeClaim phải tồn tại trong cùng namespace với Pod
tham chiếu đến nó.

Bạn có thể tham chiếu một ResourceClaim được sinh tự động trong Pod, nhưng cách này không
được khuyến nghị vì các ResourceClaim sinh tự động gắn với thời gian tồn tại của
Pod hoặc PodGroup đã kích hoạt việc sinh ra chúng.

Để tìm hiểu cách claim tài nguyên bằng một trong các phương thức này, xem
[Cấp phát thiết bị cho workload với DRA](270-allocate-devices-dra-vi.md).

#### Danh sách ưu tiên (Prioritized list) {#prioritized-list}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Bạn có thể cung cấp một danh sách ưu tiên các subrequest (yêu cầu con) cho các request trong một
ResourceClaim hoặc ResourceClaimTemplate. Scheduler sau đó sẽ chọn subrequest đầu tiên có thể
được cấp phát. Điều này cho phép người dùng chỉ định các thiết bị thay thế mà workload có thể dùng
nếu lựa chọn chính không khả dụng.

Trong ví dụ bên dưới, ResourceClaimTemplate yêu cầu một thiết bị có màu đen (black)
và kích cỡ lớn (large). Nếu không có thiết bị nào với các thuộc tính đó, Pod không thể
được lập lịch. Với tính năng danh sách ưu tiên, có thể chỉ định một phương án thay thế thứ hai,
yêu cầu hai thiết bị màu trắng (white) và kích cỡ nhỏ (small). Thiết bị lớn màu đen sẽ được
cấp phát nếu nó khả dụng. Nếu không, nhưng có sẵn hai thiết bị nhỏ màu trắng,
Pod vẫn có thể chạy được.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: prioritized-list-claim-template
spec:
  spec:
    devices:
      requests:
      - name: req-0
        firstAvailable:
        - name: large-black
          deviceClassName: resource.example.com
          selectors:
          - cel:
              expression: |-
                device.attributes["resource-driver.example.com"].color == "black" &&
                device.attributes["resource-driver.example.com"].size == "large"
        - name: small-white
          deviceClassName: resource.example.com
          selectors:
          - cel:
              expression: |-
                device.attributes["resource-driver.example.com"].color == "white" &&
                device.attributes["resource-driver.example.com"].size == "small"
          count: 2
```

Nếu Pod đủ điều kiện chạy trên nhiều node trong cluster, scheduler sẽ dùng
chỉ số (index) của subrequest được chọn từ các danh sách ưu tiên như một trong các đầu vào khi
chấm điểm từng node. Do đó các node có thể cấp phát thiết bị được yêu cầu trong subrequest
có thứ hạng cao hơn sẽ có khả năng được chọn cao hơn các node chỉ có thể cấp phát thiết bị cho
những subrequest thứ hạng thấp hơn.

Quyết định được đưa ra theo từng Pod, vì vậy nếu Pod là thành viên của một ReplicaSet hoặc
nhóm tương tự, bạn không thể trông đợi tất cả thành viên của nhóm được chọn cùng một subrequest.
Workload của bạn phải có khả năng thích ứng với điều này.

#### ResourceClaim cho Workload (Workload ResourceClaims) {#workload-resourceclaims}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi bạn tổ chức các Pod bằng
[Workload API](77-workload-api-vi.md),
bạn có thể dành riêng (reserve) ResourceClaim cho toàn bộ
PodGroup
thay vì từng Pod riêng lẻ, và sinh ResourceClaimTemplate cho một
PodGroup thay vì một Pod đơn lẻ, cho phép các Pod trong một PodGroup chia sẻ
quyền truy cập vào các thiết bị đã cấp phát cho ResourceClaim được sinh ra.

Tính năng này nhắm đến hai vấn đề:

- Danh sách `status.reservedFor` của API ResourceClaim chỉ có thể chứa 256 mục.
  Vì kube-scheduler chỉ ghi nhận từng Pod riêng lẻ vào danh sách đó, nên chỉ 256 Pod
  có thể chia sẻ một ResourceClaim. Bằng cách cho phép ghi nhận PodGroup vào
  `status.reservedFor`, số Pod có thể chia sẻ một ResourceClaim sẽ lớn hơn 256 rất nhiều.
- Các Pod chỉ có thể chia sẻ một ResourceClaim khi biết chính xác tên của nó. Với các
  workload phức tạp nhân bản (replicate) từng _nhóm_ Pod, các ResourceClaim được chia sẻ
  bởi các Pod trong mỗi nhóm cần được tạo và xóa một cách tường minh khi tập hợp
  các nhóm tăng hoặc giảm quy mô. Bằng cách sinh ResourceClaim cho mỗi PodGroup, một
  ResourceClaimTemplate duy nhất có thể là nền tảng cho các ResourceClaim vừa được
  nhân bản tự động vừa có thể chia sẻ giữa các Pod trong một PodGroup.

API PodGroup định nghĩa trường `spec.resourceClaims` với cấu trúc giống
và ý nghĩa tương tự trường `spec.resourceClaims` trong API Pod:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-group
  namespace: some-ns
spec:
  ...
  resourceClaims:
  - name: pg-claim
    resourceClaimName: my-pg-claim
  - name: pg-claim-template
    resourceClaimTemplateName: my-pg-template
```

Giống như claim do Pod tạo, claim cho PodGroup định nghĩa `resourceClaimName`
sẽ tham chiếu đến một ResourceClaim theo tên. Claim định nghĩa `resourceClaimTemplateName`
tham chiếu đến một ResourceClaimTemplate, template này nhân bản thành một ResourceClaim cho
toàn bộ PodGroup và có thể được chia sẻ giữa các Pod của nhóm.

Khi một Pod định nghĩa một claim có `name`, `resourceClaimName`, và
`resourceClaimTemplateName` cùng khớp với một trong các mục `spec.resourceClaims`
của PodGroup chứa nó, thì kube-scheduler dành riêng ResourceClaim cho
PodGroup thay vì cho Pod. Nếu claim của Pod không khớp với claim nào của
PodGroup, thì kube-scheduler dành riêng ResourceClaim cho Pod. Trong cả hai
trường hợp, việc dành riêng được ghi nhận trong `status.reservedFor` của ResourceClaim.
Các mục dành riêng của PodGroup cùng phần cấp phát tài nguyên tương ứng vẫn tồn tại trong
ResourceClaim cho đến khi PodGroup bị xóa, kể cả khi nhóm không còn
Pod nào.

Khi một claim của Pod khớp với claim của PodGroup và định nghĩa
`resourceClaimTemplateName`, thì một ResourceClaim được sinh ra cho
PodGroup. Các Pod khác trong nhóm định nghĩa cùng claim đó sẽ chia sẻ
ResourceClaim được sinh ra này thay vì kích hoạt sinh một ResourceClaim mới
cho từng Pod. Dù claim `resourceClaimTemplateName` có khớp với claim của
PodGroup hay không, tên của ResourceClaim được sinh ra đều được ghi nhận trong
`status.resourceClaimStatuses` của Pod.

Các ResourceClaim được sinh từ một ResourceClaimTemplate cho một
PodGroup tuân theo vòng đời của PodGroup đó. ResourceClaim được tạo
lần đầu khi cả PodGroup lẫn ResourceClaimTemplate của nó đều tồn tại.
ResourceClaim bị xóa sau khi PodGroup đã bị xóa và
ResourceClaim không còn được dành riêng nữa.

Xem xét ví dụ sau:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-group
  namespace: some-ns
spec:
  ...
  resourceClaims:
  - name: pg-claim
    resourceClaimName: my-pg-claim
  - name: pg-claim-template
    resourceClaimTemplateName: my-pg-template
---
apiVersion: v1
kind: Pod
metadata:
  name: training-group-pod-1
  namespace: some-ns
spec:
  ...
  schedulingGroup:
    podGroupName: training-group
  resourceClaims:
  - name: pod-claim
    resourceClaimName: my-pod-claim
  - name: pod-claim-template
    resourceClaimTemplateName: my-pod-template
  - name: pg-claim
    resourceClaimName: my-pg-claim
  - name: pg-claim-template
    resourceClaimTemplateName: my-pg-template
```

Trong ví dụ này, PodGroup `training-group` có một Pod tên `training-group-pod-1`.
Các claim `pod-claim` và `pod-claim-template` của Pod không khớp với
claim nào do PodGroup tạo, nên những claim đó không bị ảnh hưởng bởi
PodGroup: ResourceClaim `my-pod-claim` được dành riêng cho Pod và một
ResourceClaim được sinh từ ResourceClaimTemplate `my-pod-template` cũng
được dành riêng cho Pod. Các claim `pg-claim` và `pg-claim-template` khớp với
các claim do PodGroup tạo. ResourceClaim `my-pg-claim` được dành riêng cho
PodGroup và một ResourceClaim được sinh từ ResourceClaimTemplate
`my-pg-template` cũng được dành riêng cho PodGroup.

Việc liên kết ResourceClaim với các tài nguyên Workload API là một *tính năng alpha* và
chỉ được bật khi [feature gate `DRAWorkloadResourceClaims`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAWorkloadResourceClaims)
được bật trong kube-apiserver, kube-controller-manager, kube-scheduler, và kubelet.

### ResourceSlice {#resourceslice}

Mỗi ResourceSlice đại diện cho một hoặc nhiều thiết bị trong một pool (bể tài nguyên).
Pool được quản lý bởi một driver thiết bị; driver này tạo và quản lý các ResourceSlice.
Các tài nguyên trong một pool có thể được biểu diễn bởi một ResourceSlice duy nhất hoặc trải
trên nhiều ResourceSlice.

ResourceSlice cung cấp thông tin hữu ích cho người dùng thiết bị và cho scheduler,
và đóng vai trò then chốt trong cấp phát tài nguyên động. Mỗi ResourceSlice phải bao gồm
các thông tin sau:

* **Resource pool**: một nhóm gồm một hoặc nhiều tài nguyên do driver quản lý.
  Pool có thể trải trên nhiều ResourceSlice. Các thay đổi đối với tài nguyên trong một
  pool phải được lan truyền đến tất cả các ResourceSlice trong pool đó. Driver
  thiết bị quản lý pool chịu trách nhiệm đảm bảo việc lan truyền này diễn ra.
* **Devices**: các thiết bị trong pool được quản lý. Một ResourceSlice có thể liệt kê mọi
  thiết bị trong pool hoặc một tập con các thiết bị trong pool. ResourceSlice
  định nghĩa thông tin thiết bị như thuộc tính, phiên bản, và dung lượng. Người dùng
  thiết bị có thể chọn thiết bị để cấp phát bằng cách lọc theo thông tin thiết bị
  trong các ResourceClaim hoặc DeviceClass.
* **Nodes**: các node có thể truy cập các tài nguyên. Driver có thể chọn những node nào
  được truy cập tài nguyên, có thể là tất cả các node trong
  cluster, một node cụ thể theo tên, hoặc các node có những nhãn (label) node nhất định.

Driver dùng một controller để
đối chiếu (reconcile) các ResourceSlice trong cluster với thông tin mà driver cần
công bố. Controller này ghi đè mọi thay đổi thủ công, chẳng hạn khi người dùng cluster
tạo hoặc sửa đổi ResourceSlice.

Xem xét ví dụ ResourceSlice sau:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: cat-slice
spec:
  driver: "resource-driver.example.com"
  pool:
    generation: 1
    name: "black-cat-pool"
    resourceSliceCount: 1
  # Trường allNodes xác định liệu mọi node trong cluster có thể truy cập thiết bị hay không.
  allNodes: true
  devices:
  - name: "large-black-cat"
    attributes:
      color:
        string: "black"
      size:
        string: "large"
      cat:
        bool: true
```

ResourceSlice này được quản lý bởi driver `resource-driver.example.com` trong
pool `black-cat-pool`. Trường `allNodes: true` cho biết mọi node trong
cluster đều có thể truy cập các thiết bị. Có một thiết bị trong ResourceSlice, tên là
`large-black-cat`, với các thuộc tính sau:

* `color`: `black`
* `size`: `large`
* `cat`: `true`

Một DeviceClass có thể chọn ResourceSlice này bằng các thuộc tính đó, và một
ResourceClaim có thể lọc để chọn những thiết bị cụ thể trong DeviceClass đó.

#### Đặt tên và mức ưu tiên (Naming and prioritization) {#resourceslice-naming-and-prioritization}

Thứ tự mà scheduler của Kubernetes đánh giá các thiết bị để cấp phát được
xác định bởi thứ tự sắp xếp theo từ điển (lexicographical) của tên ResourceSlice và tên resource pool.
Scheduler dùng chiến lược first-fit, nghĩa là nó chọn thiết bị khả dụng đầu tiên
thỏa mãn các yêu cầu của claim.

Điều này cho phép mức ưu tiên của việc cấp phát tài nguyên được tác động bởi tên
được gán cho các pool và ResourceSlice. Lưu ý rằng các pool không có
[điều kiện gắn kết (binding conditions)](#device-binding-conditions) luôn được đánh giá trước những pool
có binding conditions, bất kể tên của chúng.

Với các driver được xây dựng bằng gói Go `k8s.io/dynamic-resources/kubeletplugin` hoặc
controller ResourceSlice từ module đó, các thành phần này tự động xử lý
việc đặt tên ResourceSlice để đảm bảo chúng được đánh giá theo thứ tự mà driver chỉ định.

## Cách hoạt động của việc cấp phát tài nguyên với DRA (How resource allocation with DRA works) {#how-it-works}

Các phần sau mô tả luồng công việc cho các
[nhóm người dùng DRA](#dra-user-types) khác nhau và cho hệ thống Kubernetes trong quá trình
cấp phát tài nguyên động.

### Luồng công việc cho người dùng (Workflow for users) {#user-workflow}

1. **Tạo driver**: chủ sở hữu thiết bị hoặc các bên thứ ba tạo các driver
   có khả năng tạo và quản lý ResourceSlice trong cluster. Các driver này
   có thể tùy chọn tạo thêm các DeviceClass định nghĩa một danh mục thiết bị và
   cách yêu cầu chúng.
1. **Cấu hình cluster**: quản trị viên cluster tạo cluster, gắn thiết bị vào
   các node, và cài đặt các driver thiết bị DRA. Quản trị viên cluster có thể tùy chọn tạo
   các DeviceClass định nghĩa các danh mục thiết bị và cách yêu cầu chúng.
1. **Resource claim**: người vận hành workload tạo các ResourceClaimTemplate hoặc
   ResourceClaim yêu cầu các cấu hình thiết bị cụ thể trong phạm vi một
   DeviceClass. Trong cùng bước này, người vận hành workload sửa các manifest Kubernetes
   của họ để yêu cầu những ResourceClaimTemplate hoặc ResourceClaim đó.

### Luồng công việc của Kubernetes (Workflow for Kubernetes) {#kubernetes-workflow}

1. **Tạo ResourceSlice**: các driver trong cluster tạo các ResourceSlice
   đại diện cho một hoặc nhiều thiết bị trong một pool các thiết bị tương tự được quản lý.
1. **Tạo workload**: control plane của cluster kiểm tra các workload mới để tìm
   các tham chiếu đến ResourceClaimTemplate hoặc đến ResourceClaim cụ thể.

   * Nếu workload dùng ResourceClaimTemplate, một controller có tên
     `resourceclaim-controller` sinh các ResourceClaim cho workload.
   * Nếu workload dùng một ResourceClaim cụ thể, Kubernetes kiểm tra liệu
     ResourceClaim đó có tồn tại trong cluster hay không. Nếu ResourceClaim không
     tồn tại, các Pod sẽ không được triển khai.

1. **Lọc ResourceSlice**: với mỗi Pod, Kubernetes kiểm tra các
   ResourceSlice trong cluster để tìm một thiết bị thỏa mãn tất cả các
   tiêu chí sau:

   * Các node có thể truy cập tài nguyên đủ điều kiện để chạy Pod.
   * ResourceSlice có tài nguyên chưa được cấp phát khớp với các yêu cầu của
     ResourceClaim của Pod.

1. **Cấp phát tài nguyên**: sau khi tìm được một ResourceSlice đủ điều kiện cho
   ResourceClaim của Pod, scheduler của Kubernetes cập nhật ResourceClaim
   với các chi tiết cấp phát. Scheduler dùng chiến lược first-fit và
   đánh giá các pool và ResourceSlice theo thứ tự từ điển của tên chúng.
   Driver có thể ưu tiên các slice hoặc pool cụ thể bằng cách đặt tên phù hợp.
   Xem chi tiết ở [Đặt tên và mức ưu tiên](#resourceslice-naming-and-prioritization).
1. **Lập lịch Pod**: khi việc cấp phát tài nguyên hoàn tất, scheduler
   đặt Pod lên một node có thể truy cập tài nguyên đã được cấp phát. Driver
   thiết bị và kubelet trên node đó cấu hình thiết bị và quyền truy cập
   của Pod tới thiết bị.

## Khả năng quan sát tài nguyên động (Observability of dynamic resources) {#observability-dynamic-resources}

Bạn có thể kiểm tra trạng thái của các tài nguyên được cấp phát động bằng bất kỳ
phương pháp nào sau đây:

* [Metric thiết bị của kubelet](#monitoring-resources)
* [Trạng thái của ResourceClaim](#resourceclaim-device-status)
* [Giám sát sức khỏe thiết bị](#device-health-monitoring)

### Metric thiết bị của kubelet (kubelet device metrics) {#monitoring-resources}

Dịch vụ gRPC `PodResourcesLister` của kubelet cho phép bạn giám sát các thiết bị đang được sử dụng.
Thông điệp `DynamicResource` cung cấp thông tin đặc thù cho cấp phát tài nguyên
động, chẳng hạn tên thiết bị và tên claim. Xem chi tiết ở
[Giám sát tài nguyên device plugin](184-device-plugins-vi.md#monitoring-device-plugin-resources).

### Trạng thái thiết bị trong ResourceClaim (ResourceClaim device status) {#resourceclaim-device-status}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [beta]`

Các driver DRA có thể báo cáo dữ liệu
[trạng thái thiết bị](16-working-with-objects-vi.md#object-spec-and-status)
đặc thù theo driver cho từng thiết bị đã cấp phát trong trường `status.devices` của một ResourceClaim.
Ví dụ, driver có thể liệt kê các địa chỉ IP được gán cho một
thiết bị network interface. Việc cập nhật trường này yêu cầu các quyền RBAC tổng hợp (synthetic) cụ thể,
xem
[Hướng dẫn tăng cường bảo mật - Dynamic Resource Allocation](125-hardening-dra-vi.md)
và
[Tăng cường bảo mật Dynamic Resource Allocation trong cluster của bạn](211-hardening-dra-tasks-vi.md).

Độ chính xác của thông tin mà driver thêm vào trường `status.devices` của
ResourceClaim phụ thuộc vào driver. Hãy đánh giá các driver để quyết định liệu
bạn có thể tin cậy trường này như nguồn thông tin thiết bị duy nhất hay không.

Nếu bạn tắt
[feature gate `DRAResourceClaimDeviceStatus`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAResourceClaimDeviceStatus),
trường `status.devices` sẽ tự động bị xóa khi lưu ResourceClaim.
Trạng thái thiết bị của ResourceClaim được hỗ trợ khi driver DRA
có thể cập nhật một ResourceClaim hiện có mà trường `status.devices`
đã được thiết lập.

Xem chi tiết về trường `status.devices` trong tài liệu tham khảo API
[ResourceClaim](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/resource-claim-v1/#ResourceClaimStatus).

### Giám sát sức khỏe thiết bị (Device Health Monitoring) {#device-health-monitoring}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Kubernetes cung cấp một cơ chế để giám sát và báo cáo sức khỏe (health) của các tài nguyên hạ tầng được cấp phát động.
Với các ứng dụng có trạng thái (stateful) chạy trên phần cứng chuyên dụng, việc biết khi nào một thiết bị bị lỗi hoặc trở nên không khỏe mạnh là rất quan trọng. Việc biết được thiết bị có phục hồi hay không cũng rất hữu ích.

Để dùng chức năng này, [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/resource-health-status/) `ResourceHealthStatus` phải được bật (beta và được bật mặc định từ v1.36), và driver DRA phải triển khai dịch vụ gRPC `DRAResourceHealth`.

Khi một driver DRA phát hiện một thiết bị đã cấp phát trở nên không khỏe mạnh, nó báo cáo trạng thái này về kubelet. Thông tin sức khỏe này sau đó được hiển thị trực tiếp trong trạng thái (status) của Pod. Kubelet điền vào trường `allocatedResourcesStatus` trong status của từng container, mô tả chi tiết sức khỏe của mỗi thiết bị được gán cho container đó. Mỗi mục sức khỏe tài nguyên có thể bao gồm một trường `message` tùy chọn chứa thêm ngữ cảnh dễ đọc về trạng thái sức khỏe, chẳng hạn chi tiết lỗi hoặc lý do hỏng hóc.

Nếu kubelet không nhận được cập nhật sức khỏe từ driver DRA trong một khoảng thời gian chờ (timeout), trạng thái sức khỏe của thiết bị được đánh dấu là "Unknown". Các driver DRA có thể cấu hình timeout này theo từng thiết bị bằng cách đặt trường `health_check_timeout_seconds` trong thông điệp gRPC `DeviceHealth`. Nếu không chỉ định, kubelet dùng timeout mặc định 30 giây. Điều này cho phép các loại phần cứng khác nhau (ví dụ GPU, FPGA, hoặc thiết bị lưu trữ) dùng các giá trị timeout phù hợp với đặc điểm báo cáo sức khỏe của chúng.

Cơ chế này cung cấp khả năng quan sát quan trọng để người dùng và các controller phản ứng với các hỏng hóc phần cứng.
Với một Pod đang gặp lỗi, bạn có thể xem trạng thái này để xác định liệu lỗi có liên quan đến một thiết bị không khỏe mạnh hay không.

> **Ghi chú:**
> Trạng thái sức khỏe thiết bị không được cập nhật trong status của Pod sau khi Pod đã kết thúc (ví dụ, ở trạng thái Failed).

## Pod được lập lịch sẵn (Pre-scheduled Pods)

Khi bạn — hoặc một API client khác — tạo một Pod với `spec.nodeName` đã được thiết lập sẵn, scheduler bị bỏ qua.
Nếu một ResourceClaim nào đó mà Pod cần chưa tồn tại, chưa được cấp phát
hoặc chưa được dành riêng cho Pod, thì kubelet sẽ không chạy được Pod và
sẽ kiểm tra lại định kỳ vì các yêu cầu đó vẫn có thể được đáp ứng sau này.

Tình huống như vậy cũng có thể phát sinh khi tính năng cấp phát tài nguyên động
chưa được bật trong scheduler tại thời điểm Pod được lập lịch
(chênh lệch phiên bản, cấu hình, feature gate, v.v.). kube-controller-manager
phát hiện điều này và cố gắng làm cho Pod chạy được bằng cách dành riêng các
ResourceClaim cần thiết. Tuy nhiên, việc này chỉ hoạt động nếu những claim đó đã được
scheduler cấp phát cho một Pod khác.

Tốt hơn là tránh bỏ qua scheduler, vì một Pod đã được gán vào node sẽ
chiếm giữ các tài nguyên thông thường (RAM, CPU) mà sau đó không thể dùng cho các Pod khác
trong lúc Pod bị kẹt. Để một Pod chạy trên một node cụ thể mà vẫn đi
qua luồng lập lịch bình thường, hãy tạo Pod với một node selector
khớp chính xác với node mong muốn:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-cats
spec:
  nodeSelector:
    kubernetes.io/hostname: name-of-the-intended-node
  ...
```

Bạn cũng có thể can thiệp (mutate) Pod đang được tạo, tại thời điểm admission, để bỏ
trường `.spec.nodeName` và dùng node selector thay thế.

## Hạn chế (Limitations)

* Scheduler của Kubernetes không hỗ trợ
  [chiếm chỗ (preemption)](141-pod-priority-preemption-vi.md) cho
  các tài nguyên DRA. Điều này nghĩa là một Pod hiện có đang chạy trên node và đang
  dùng tài nguyên DRA không thể bị chiếm chỗ bởi một Pod có độ ưu tiên cao hơn cũng cần
  tài nguyên DRA. Pod có độ ưu tiên cao sẽ ở trạng thái pending cho đến khi thiết bị
  trở nên khả dụng, tức là khi Pod xung đột kết thúc hoặc bị
  xóa thủ công.

## Các tính năng beta của DRA (DRA beta features) {#beta-features}

Các phần sau mô tả những tính năng DRA hỗ trợ các trường hợp sử dụng
nâng cao. Việc dùng chúng là tùy chọn và có thể chỉ liên quan đến các driver DRA
hỗ trợ chúng.

Một số tính năng đang ở [giai đoạn tính năng](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages)
Alpha hoặc Beta.
Chúng phụ thuộc vào các feature gate và có thể phụ thuộc vào các
nhóm API (API group) bổ sung.
Xem thêm thông tin ở
[Thiết lập DRA trong cluster](271-set-up-dra-cluster-vi.md).

### Quyền truy cập quản trị (Admin access) {#admin-access}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Bạn có thể đánh dấu một request trong ResourceClaim hoặc ResourceClaimTemplate là có
các đặc quyền dành cho công việc bảo trì và khắc phục sự cố. Một request với
quyền truy cập quản trị cấp quyền truy cập vào các thiết bị đang được sử dụng và có thể kích hoạt thêm
quyền hạn khi đưa thiết bị vào sử dụng trong container:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: large-black-cat-claim-template
spec:
  spec:
    devices:
      requests:
      - name: req-0
        exactly:
          deviceClassName: resource.example.com
          allocationMode: All
          adminAccess: true
```

Quyền truy cập quản trị là chế độ đặc quyền và không nên được cấp cho người dùng thông thường trong
các cluster đa người thuê (multi-tenant). Chỉ những người dùng được ủy quyền
tạo các đối tượng ResourceClaim hoặc ResourceClaimTemplate trong các namespace có nhãn
`resource.kubernetes.io/admin-access: "true"` (phân biệt chữ hoa/thường) mới có thể dùng
trường `adminAccess`. Điều này đảm bảo người dùng không phải quản trị viên không thể lạm dụng
tính năng.

Quyền truy cập quản trị là một *tính năng beta* và được bật mặc định với
[feature gate `DRAAdminAccess`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAAdminAccess)
trong kube-apiserver, kube-scheduler, và kubelet.

### Ủy quyền trạng thái chi tiết (Granular status authorization) {#granular-status-authorization}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Bắt đầu từ Kubernetes v1.36, DRA thực thi các kiểm tra ủy quyền chi tiết cho các cập nhật
vào status của `ResourceClaim` bằng cách dùng các subresource tổng hợp (synthetic) và các verb nhận biết node (node-aware).

Để có hướng dẫn tăng cường bảo mật, bao gồm các ví dụ RBAC cho scheduler và các driver
DRA, xem
[Hướng dẫn tăng cường bảo mật - Dynamic Resource Allocation](125-hardening-dra-vi.md).

Để có quy trình từng bước dành cho quản trị viên cluster, xem
[Tăng cường bảo mật Dynamic Resource Allocation trong cluster của bạn](211-hardening-dra-tasks-vi.md).

## Các tính năng alpha của DRA (DRA alpha features) {#alpha-features}

Các phần sau mô tả những tính năng DRA đang ở [giai đoạn tính năng](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages)
Alpha.
Chúng phụ thuộc vào việc bật các feature gate và có thể phụ thuộc vào các
nhóm API bổ sung.
Xem thêm thông tin ở
[Thiết lập DRA trong cluster](271-set-up-dra-cluster-vi.md).

### Cấp phát extended resource bằng DRA (Extended resource allocation by DRA) {#extended-resource}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Bạn có thể cung cấp một tên extended resource cho một DeviceClass. Scheduler sau đó sẽ
chọn các thiết bị khớp với class đó cho các yêu cầu extended resource.
Điều này cho phép người dùng tiếp tục dùng các yêu cầu extended resource trong Pod để yêu cầu
hoặc extended resource do device plugin cung cấp, hoặc các thiết bị DRA.
Cùng một extended resource có thể được cung cấp bởi device plugin, hoặc bởi DRA trên một node duy nhất của cluster.
Cùng một extended resource có thể được cung cấp bởi device plugin trên một số node, và bởi DRA trên các node khác trong cùng cluster.

Trong ví dụ bên dưới, DeviceClass được gán extendedResourceName `example.com/gpu`.
Nếu một Pod yêu cầu extended resource `example.com/gpu: 2`, nó có thể được lập lịch lên
một node có hai thiết bị trở lên khớp với DeviceClass đó.

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.example.com
spec:
  selectors:
  - cel:
      expression: device.driver == 'gpu.example.com' && device.attributes['gpu.example.com'].type
        == 'gpu'
  extendedResourceName: example.com/gpu
```

Ngoài ra, người dùng có thể dùng một extended resource đặc biệt để cấp phát thiết bị mà không
cần tạo ResourceClaim một cách tường minh, bằng cách dùng tiền tố tên extended resource
`deviceclass.resource.kubernetes.io/` kèm theo tên DeviceClass.
Cách này hoạt động với bất kỳ DeviceClass nào, kể cả khi nó không chỉ định tên extended resource.
ResourceClaim thu được sẽ chứa một request với `ExactCount` bằng
số lượng thiết bị được chỉ định của DeviceClass đó.

Cấp phát extended resource bằng DRA là một *tính năng beta* và được bật mặc định với
[feature gate `DRAExtendedResource`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAExtendedResource)
trong kube-apiserver, kube-scheduler, kube-controller-manager, và kubelet.

### Thiết bị có thể phân vùng (Partitionable devices) {#partitionable-devices}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Các thiết bị được biểu diễn trong DRA không nhất thiết phải là một đơn vị đơn lẻ kết nối với một máy duy nhất,
mà cũng có thể là một thiết bị logic được tạo thành từ nhiều thiết bị kết nối với nhiều máy.
Các thiết bị này có thể tiêu thụ những tài nguyên chồng lấn của các thiết bị vật lý bên dưới,
nghĩa là khi một thiết bị logic được cấp phát thì các thiết bị khác sẽ không còn khả dụng nữa.

Trong API ResourceSlice, điều này được biểu diễn dưới dạng một danh sách các CounterSet có tên, mỗi CounterSet
chứa một tập các counter (bộ đếm) có tên. Các counter đại diện cho các tài nguyên khả dụng trên thiết bị
vật lý mà các thiết bị logic được công bố qua DRA sử dụng.

Các thiết bị logic có thể chỉ định danh sách ConsumesCounters. Mỗi mục chứa một tham chiếu đến một CounterSet
và một tập các counter có tên cùng với lượng chúng sẽ tiêu thụ. Do đó, để một thiết bị có thể cấp phát được,
các counter set được tham chiếu phải có đủ số lượng cho các counter mà thiết bị đó tham chiếu.

CounterSet phải được chỉ định trong các ResourceSlice tách biệt với các thiết bị.
Thiết bị có thể tiêu thụ counter từ bất kỳ CounterSet nào được định nghĩa trong cùng resource pool với thiết bị.

Đây là ví dụ về hai thiết bị, mỗi thiết bị tiêu thụ 6Gi bộ nhớ từ một counter dùng chung có 8Gi bộ nhớ.
Do vậy, chỉ một trong hai thiết bị có thể được cấp phát tại mỗi thời điểm.
Scheduler xử lý điều này một cách trong suốt đối với bên sử dụng, vì API ResourceClaim không bị ảnh hưởng.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: resourceslice-with-countersets
spec:
  nodeName: worker-1
  pool:
    name: pool
    generation: 1
    resourceSliceCount: 2
  driver: dra.example.com
  sharedCounters:
  - name: gpu-1-counters
    counters:
      memory:
        value: 8Gi
---
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: resourceslice-with-devices
spec:
  nodeName: worker-1
  pool:
    name: pool
    generation: 1
    resourceSliceCount: 2
  driver: dra.example.com
  devices:
  - name: device-1
    consumesCounters:
    - counterSet: gpu-1-counters
      counters:
        memory:
          value: 6Gi
  - name: device-2
    consumesCounters:
    - counterSet: gpu-1-counters
      counters:
        memory:
          value: 6Gi
```

Thiết bị có thể phân vùng là một *tính năng beta* và được bật khi
[feature gate `DRAPartitionableDevices`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAPartitionableDevices)
được giữ ở trạng thái bật trong kube-apiserver và kube-scheduler.

### Dung lượng tiêu thụ được (Consumable capacity) {#consumable-capacity}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Tính năng consumable capacity cho phép cùng một thiết bị được tiêu thụ bởi nhiều ResourceClaim độc lập,
trong đó scheduler của Kubernetes quản lý lượng dung lượng của thiết bị mà mỗi claim sử dụng.
Điều này tương tự cách các Pod có thể chia sẻ tài nguyên trên một Node; các ResourceClaim có thể chia sẻ tài nguyên trên một Device.

Driver thiết bị có thể đặt trường `allowMultipleAllocations` được thêm vào `.spec.devices` của `ResourceSlice`
để cho phép cấp phát thiết bị đó cho nhiều ResourceClaim độc lập hoặc cho nhiều request trong cùng một ResourceClaim.

Người dùng có thể đặt trường `capacity` được thêm vào `spec.devices.requests` của `ResourceClaim` để chỉ định yêu cầu tài nguyên thiết bị cho mỗi lần cấp phát.

Với thiết bị cho phép nhiều lần cấp phát, dung lượng được yêu cầu sẽ được trích ra — hay tiêu thụ — từ tổng dung lượng của nó,
một khái niệm gọi là **consumable capacity** (dung lượng tiêu thụ được).
Sau đó, scheduler đảm bảo tổng dung lượng đã tiêu thụ trên tất cả các claim không vượt quá dung lượng tổng thể của thiết bị.
Hơn nữa, tác giả driver có thể dùng ràng buộc `requestPolicy` trên từng dung lượng cụ thể của thiết bị để kiểm soát
cách các dung lượng đó được tiêu thụ.
Ví dụ, tác giả driver có thể chỉ định rằng một dung lượng nhất định chỉ được tiêu thụ theo các bậc tăng 1Gi.

Đây là ví dụ về một thiết bị mạng cho phép nhiều lần cấp phát và có dung lượng băng thông tiêu thụ được.

```yaml
kind: ResourceSlice
apiVersion: resource.k8s.io/v1
metadata:
  name: resourceslice
spec:
  nodeName: worker-1
  pool:
    name: pool
    generation: 1
    resourceSliceCount: 1
  driver: dra.example.com
  devices:
  - name: eth1
    allowMultipleAllocations: true
    attributes:
      name:
        string: "eth1"
    capacity:
      bandwidth:
        requestPolicy:
          default: "1M"
          validRange:
            min: "1M"
            step: "8"
        value: "10G"
```

Dung lượng tiêu thụ được có thể được yêu cầu như trong ví dụ bên dưới.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: bandwidth-claim-template
spec:
  spec:
    devices:
      requests:
      - name: req-0
        exactly:
          deviceClassName: resource.example.com
          capacity:
            requests:
              bandwidth: 1G
```

Kết quả cấp phát sẽ bao gồm dung lượng đã tiêu thụ và định danh (identifier) của phần chia sẻ.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
...
status:
  allocation:
    devices:
      results:
      - consumedCapacity:
          bandwidth: 1G
        device: eth1
        shareID: "a671734a-e8e5-11e4-8fde-42010af09327"
```

Trong ví dụ này, một thiết bị có thể cấp phát nhiều lần đã được chọn. Tuy nhiên, bất kỳ thiết bị `resource.example.com` nào
có ít nhất 1G băng thông theo yêu cầu đều có thể đáp ứng yêu cầu này.
Nếu một thiết bị không thể cấp phát nhiều lần được chọn, việc cấp phát sẽ dẫn đến toàn bộ thiết bị.
Để buộc chỉ dùng các thiết bị có thể cấp phát nhiều lần, bạn có thể dùng tiêu chí CEL `device.allowMultipleAllocations == true`.

#### Ràng buộc DistinctAttribute (DistinctAttribute constraint)

Khi yêu cầu nhiều thiết bị trong một ResourceClaim, bạn có thể dùng ràng buộc DistinctAttribute
để đảm bảo mỗi thiết bị được cấp phát có giá trị khác nhau cho một thuộc tính
được chỉ định. Ràng buộc này được giới thiệu cùng với tính năng consumable capacity.

Ràng buộc DistinctAttribute đặc biệt hữu ích khi làm việc với
các thiết bị có thể cấp phát nhiều lần. Nó ngăn scheduler cấp phát cùng một
thiết bị nhiều lần trong một ResourceClaim duy nhất, kể cả khi thiết bị đó cho phép
nhiều lần cấp phát.

Ngoài việc ngăn cấp phát trùng lặp, ràng buộc này giúp tối ưu hiệu năng
bằng cách đảm bảo các thiết bị được phân bổ dựa trên thuộc tính của chúng. Ví dụ, bạn có thể
dùng nó để phân bổ thiết bị trên các NUMA node khác nhau nhằm tối ưu băng thông bộ nhớ
và giảm tranh chấp.

### Taint và toleration cho thiết bị (Device taints and tolerations) {#device-taints-and-tolerations}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Taint của thiết bị tương tự taint của node: một taint có key dạng chuỗi, value dạng chuỗi, và một effect.
Effect được áp dụng cho ResourceClaim đang dùng thiết bị bị taint và cho tất cả các Pod tham chiếu ResourceClaim đó.
Effect "NoSchedule" ngăn việc lập lịch các Pod đó.
Các thiết bị bị taint bị bỏ qua khi cấp phát một ResourceClaim, vì việc dùng chúng sẽ ngăn cản việc lập lịch Pod.

Effect "NoExecute" bao hàm "NoSchedule" và ngoài ra còn gây ra việc trục xuất (eviction) tất cả các Pod
đã được lập lịch trước đó.
Việc trục xuất này được triển khai trong controller trục xuất theo device taint (device taint eviction controller) trong kube-controller-manager bằng cách xóa các Pod bị ảnh hưởng.

Effect "None" được scheduler và controller trục xuất bỏ qua.
Các driver DRA có thể dùng nó để thông báo các ngoại lệ cho quản trị viên hoặc các controller khác,
ví dụ như tình trạng sức khỏe suy giảm của một thiết bị. Quản trị viên cũng có thể dùng nó để
chạy thử (dry-run) việc trục xuất Pod trong DeviceTaintRule (chi tiết hơn ở bên dưới).

ResourceClaim có thể tolerate (chấp nhận) các taint. Nếu một taint được tolerate, effect của nó không được áp dụng.
Một toleration rỗng khớp với mọi taint. Một toleration có thể được giới hạn cho những effect nhất định
và/hoặc khớp với các cặp key/value nhất định.
Một toleration có thể kiểm tra rằng một key nhất định tồn tại, bất kể giá trị của nó là gì, hoặc có thể kiểm tra
các giá trị cụ thể của một key.
Xem thêm thông tin về cách khớp này ở
[các khái niệm về taint của node](139-taint-and-toleration-vi.md#concepts).

Việc trục xuất có thể được trì hoãn bằng cách tolerate một taint trong một khoảng thời gian nhất định.
Khoảng trì hoãn đó bắt đầu tại thời điểm taint được thêm vào thiết bị, thời điểm này được ghi lại trong một trường của taint.

Các taint cũng được áp dụng như mô tả ở trên đối với các ResourceClaim cấp phát "tất cả" thiết bị trên một node.
Tất cả thiết bị phải không bị taint hoặc tất cả các taint của chúng phải được tolerate.
Việc cấp phát thiết bị với quyền truy cập quản trị (mô tả [ở trên](#admin-access))
cũng không được miễn trừ. Quản trị viên dùng chế độ đó phải tolerate tường minh tất cả các taint
để truy cập các thiết bị bị taint.

Taint và toleration cho thiết bị là một *tính năng beta* và được bật khi
[feature gate `DRADeviceTaints`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRADeviceTaints)
được giữ ở trạng thái bật trong kube-apiserver, kube-controller-manager và kube-scheduler.
Để dùng DeviceTaintRule, phiên bản API `resource.k8s.io/v1beta2` phải được
bật cùng với [feature gate `DRADeviceTaintRules`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRADeviceTaintRules).
Khác với `DRADeviceTaints`, `DRADeviceTaintRules` mặc định bị tắt do sự phụ thuộc
vào nhóm API beta này, vốn phải bị tắt theo mặc định.

Bạn có thể thêm taint cho thiết bị theo các cách sau, bằng cách dùng kind API DeviceTaintRule.

#### Taint do driver đặt (Taints set by the driver)

Một driver DRA có thể thêm taint vào thông tin thiết bị mà nó công bố trong các ResourceSlice.
Tham khảo tài liệu của driver DRA để biết liệu driver có dùng taint hay không và các key, value của chúng là gì.

#### Taint do quản trị viên đặt (Taints set by an admin)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Quản trị viên hoặc một thành phần control plane có thể taint các thiết bị mà không cần yêu cầu
driver DRA đưa taint vào thông tin thiết bị của nó trong các ResourceSlice.
Họ làm điều đó bằng cách tạo các DeviceTaintRule.
Mỗi DeviceTaintRule thêm một taint vào các thiết bị khớp với device selector.
Nếu không có selector như vậy, không thiết bị nào bị taint.
Điều này khiến việc vô tình trục xuất tất cả các Pod đang dùng ResourceClaim do quên selector trở nên khó xảy ra hơn.

Có thể chọn thiết bị bằng cách cung cấp tên của một DeviceClass, driver, pool, và/hoặc thiết bị.
DeviceClass chọn tất cả các thiết bị được chọn bởi các selector trong DeviceClass đó.
Chỉ với tên driver, quản trị viên có thể taint tất cả các thiết bị do driver đó quản lý,
ví dụ trong khi thực hiện bảo trì driver đó trên toàn bộ cluster.
Việc thêm tên pool có thể giới hạn taint vào một node duy nhất, nếu driver quản lý các thiết bị cục bộ theo node.

Cuối cùng, việc thêm tên thiết bị có thể chọn một thiết bị cụ thể.
Tên thiết bị và tên pool cũng có thể được dùng riêng lẻ, nếu muốn.
Ví dụ, các driver cho thiết bị cục bộ theo node được khuyến khích dùng tên node làm tên pool.
Khi đó, taint theo tên pool đó sẽ tự động taint tất cả thiết bị trên một node.

Driver có thể dùng các tên ổn định như "gpu-0" mà che giấu thiết bị cụ thể nào hiện đang được gán cho tên đó.
Để hỗ trợ taint một phần cứng cụ thể, có thể dùng các CEL selector trong DeviceTaintRule
để khớp với một thuộc tính ID duy nhất đặc thù của nhà cung cấp, nếu driver hỗ trợ thuộc tính như vậy cho phần cứng của nó.

Taint được áp dụng chừng nào DeviceTaintRule còn tồn tại.
Nó có thể được sửa đổi và gỡ bỏ bất kỳ lúc nào.
Đây là một ví dụ về DeviceTaintRule cho một driver DRA giả định:

```yaml
apiVersion: resource.k8s.io/v1beta2
kind: DeviceTaintRule
metadata:
  name: example
spec:
  # Toàn bộ hệ thống phần cứng của driver
  # cụ thể này đã hỏng.
  # Trục xuất tất cả các Pod và không lập lịch Pod mới.
  deviceSelector:
    driver: dra.example.com
  taint:
    key: dra.example.com/unhealthy
    value: Broken
    effect: NoExecute
```

Kube-apiserver tự động theo dõi thời điểm taint này được tạo bằng cách đặt
trường `timeAdded` trong `spec`. Khoảng thời gian toleration bắt đầu từ mốc thời gian
đó. Trong các lần cập nhật làm thay đổi effect (xem luồng trục xuất mô phỏng
bên dưới), kube-apiserver tự động cập nhật mốc thời gian. Người dùng có thể kiểm soát
mốc thời gian một cách tường minh bằng cách đặt trường này khi tạo DeviceTaintRule và
bằng cách đổi nó sang một giá trị khác khi cập nhật.

Phần status chứa một condition do controller trục xuất thêm vào:

```
kubectl describe devicetaintrules
```

```
Name:         example
...
Spec:
  Device Selector:
    Driver:  dra.example.com
  Taint:
    Effect:      NoExecute
    Key:         dra.example.com/unhealthy
    Time Added:  2025-11-05T18:15:37Z
    Value:       Broken
Status:
  Conditions:
    Last Transition Time:  2025-11-05T18:15:37Z
    Message:               1 pod evicted since starting the controller.
    Observed Generation:   1
    Reason:                Completed
    Status:                False
    Type:                  EvictionInProgress
Events:                    <none>
```

Các Pod được trục xuất bằng cách xóa chúng. Thông thường việc này diễn ra rất nhanh,
trừ khi một toleration cho taint trì hoãn nó trong một khoảng thời gian nhất định hoặc
khi có quá nhiều Pod cần trục xuất. Khi việc này mất nhiều thời gian hơn,
message cung cấp thông tin về trạng thái hiện tại:

    2 pods need to be evicted in 2 different namespaces. 1 pod evicted since starting the controller.

Condition này có thể được dùng để kiểm tra liệu một đợt trục xuất có đang diễn ra hay không:

    kubectl wait --for=condition=EvictionInProgress=false DeviceTaintRule/example

Hãy đề phòng khả năng xảy ra race giữa scheduler và controller khi quan sát taint mới
tại các thời điểm khác nhau, điều này có thể dẫn đến việc các Pod vẫn được lập lịch tại
thời điểm controller cho rằng không còn Pod nào cần trục xuất
và do đó đặt condition này thành `False`. Trên thực tế, race này được làm cho rất
khó xảy ra bằng cách chỉ cập nhật status sau một khoảng trễ có chủ đích vài
giây.

Với `effect: None`, message cung cấp thông tin về số lượng
thiết bị bị ảnh hưởng, bao nhiêu trong số đó đã được cấp phát, và bao nhiêu Pod sẽ bị
trục xuất nếu effect là `NoExecute`. Điều này có thể được dùng để chạy thử (dry-run) trước khi
thực sự kích hoạt trục xuất:

- Tạo một DeviceTaintRule với các selector mong muốn và `effect: None`.

- Xem lại message:

  ```
  3 published devices selected. 1 allocated device selected.
  1 pod would be evicted in 1 namespace if the effect was NoExecute.
  This information will not be updated again. Recreate the DeviceTaintRule to trigger an update.
  ```

  Thiết bị đã công bố (published) là những thiết bị được liệt kê trong các ResourceSlice. Taint chúng
  sẽ ngăn cấp phát cho các Pod mới. Chỉ những thiết bị đã cấp phát mới gây ra
  trục xuất các Pod đang dùng chúng.

- Sửa DeviceTaintRule và đổi effect thành `NoExecute`.

### Trạng thái resource pool (Resource pool status) {#resource-pool-status}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Bạn có thể truy vấn mức khả dụng của các thiết bị trong các resource pool bằng
API ResourcePoolStatusRequest. API này cung cấp khả năng quan sát xem có bao nhiêu thiết bị
đang khả dụng, đã cấp phát, hoặc không khả dụng trên các resource pool DRA của cluster.

Để kiểm tra trạng thái resource pool:

1. Tạo một ResourcePoolStatusRequest chỉ định tên driver (bắt buộc) và
   tùy chọn một giới hạn số pool được trả về. Bạn cũng có thể giới hạn vào một pool duy nhất bằng cách chỉ định tên pool:

   ```yaml
   apiVersion: resource.k8s.io/v1beta2
   kind: ResourcePoolStatusRequest
   metadata:
     name: check-gpus
   spec:
     driver: example.com/gpu
     # Tùy chọn: lọc theo một pool cụ thể
     # poolName: my-pool
     # Tùy chọn: giới hạn số pool được trả về (mặc định: 100, tối đa: 1000)
     # limit: 10
   ```

1. Chờ controller xử lý yêu cầu:

   ```shell
   kubectl wait --for=condition=Complete resourcepoolstatusrequest/check-gpus --timeout=30s
   ```

1. Đọc status để xem mức khả dụng của pool:

   ```shell
   kubectl get resourcepoolstatusrequest/check-gpus -o yaml
   ```

   Status bao gồm:
   - `poolCount`: tổng số pool khớp với bộ lọc (có thể lớn hơn số
     pool được liệt kê nếu bị cắt bớt bởi giới hạn).
   - `pools`: danh sách chi tiết các pool, mỗi mục chứa:
     - `driver` và `poolName`: định danh pool.
     - `generation`: thế hệ (generation) pool mới nhất được quan sát trên các ResourceSlice.
     - `resourceSliceCount`: số ResourceSlice tạo nên pool.
     - `totalDevices`: tổng số thiết bị trong pool.
     - `allocatedDevices`: các thiết bị hiện đang được cấp phát cho các claim.
     - `availableDevices`: các thiết bị khả dụng để cấp phát
       (totalDevices - allocatedDevices - unavailableDevices).
     - `unavailableDevices`: các thiết bị không khả dụng do taint hoặc các điều kiện khác.
     - `nodeName`: node gắn liền với pool, nếu có.
     - `validationError`: được đặt khi dữ liệu của pool không thể được xác thực đầy đủ
       (ví dụ trong quá trình chuyển đổi generation). Khi được đặt, các trường đếm thiết bị
       có thể không được thiết lập.
   - `conditions`: bao gồm các loại condition `Complete` (thành công) hoặc `Failed` (lỗi).

1. Xóa yêu cầu khi đã xong:

   ```shell
   kubectl delete resourcepoolstatusrequest/check-gpus
   ```

Các đối tượng ResourcePoolStatusRequest được xử lý một lần bởi một controller trong
kube-controller-manager. Phần spec là bất biến (immutable) sau khi tạo, và toàn bộ
đối tượng trở nên bất biến khi status đã được điền. Để lấy dữ liệu khả dụng
mới hơn, hãy xóa và tạo lại yêu cầu. Các yêu cầu đã hoàn thành được
tự động dọn dẹp sau 1 giờ.

Tính năng này yêu cầu quyền RBAC tường minh trên tài nguyên ResourcePoolStatusRequest.
Không có ClusterRole mặc định nào bao gồm quyền này.

Trạng thái resource pool là một *tính năng alpha* và chỉ được bật khi
[feature gate `DRAResourcePoolStatus`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRAResourcePoolStatus)
được bật trong kube-apiserver và kube-controller-manager.

### Điều kiện gắn kết thiết bị (Device binding conditions) {#device-binding-conditions}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Device Binding Conditions cho phép scheduler của Kubernetes trì hoãn việc gắn kết (binding) Pod cho đến khi
các tài nguyên bên ngoài, chẳng hạn GPU gắn qua fabric hoặc FPGA có thể lập trình lại, được xác nhận
là sẵn sàng.

Hành vi chờ này được triển khai trong
[pha PreBind](147-scheduling-framework-vi.md#pre-bind)
của scheduling framework.
Trong pha này, scheduler kiểm tra liệu tất cả các điều kiện thiết bị bắt buộc đã được
thỏa mãn hay chưa trước khi tiến hành binding.

Điều này cải thiện độ tin cậy của việc lập lịch bằng cách tránh binding quá sớm và cho phép phối hợp
với các controller thiết bị bên ngoài.

Để dùng tính năng này, các driver thiết bị (thường do chủ sở hữu driver quản lý) phải công bố
các trường sau trong phần `Device` của một `ResourceSlice`. Quản trị viên cluster
phải bật các feature gate `DRADeviceBindingConditions` và `DRAResourceClaimDeviceStatus`
để scheduler tôn trọng các trường này.

`bindingConditions`
: Danh sách các _loại condition_ phải được đặt là True (trong trường `.status.conditions` của ResourceClaim liên quan) trước khi Pod có thể được bind. Các condition này thường đại diện cho tín hiệu sẵn sàng, chẳng hạn DeviceAttached hoặc DeviceInitialized.

`bindingFailureConditions`
: Danh sách các loại condition mà nếu được đặt là True trong
  trường status.conditions của ResourceClaim liên quan sẽ biểu thị trạng thái lỗi.
  Nếu bất kỳ condition nào trong số này là True, scheduler sẽ hủy binding và lập lịch lại Pod.

`bindsToNode`
: nếu được đặt là `true`, scheduler ghi lại tên node đã chọn vào
  trường `status.allocation.nodeSelector` của ResourceClaim.
  Điều này không ảnh hưởng đến `spec.nodeSelector` của Pod. Thay vào đó, nó đặt một node selector
  bên trong ResourceClaim, mà các controller bên ngoài có thể dùng để thực hiện các thao tác
  đặc thù theo node như gắn hoặc chuẩn bị thiết bị.

Tất cả các loại condition được liệt kê trong bindingConditions và bindingFailureConditions được đánh giá
từ trường `status.conditions` của ResourceClaim.
Các controller bên ngoài chịu trách nhiệm cập nhật những condition này theo ngữ nghĩa condition
chuẩn của Kubernetes (`type`, `status`, `reason`, `message`, `lastTransitionTime`).

Scheduler chờ tối đa **600 giây** (mặc định) để tất cả các `bindingConditions` trở thành `True`.
Nếu hết thời gian chờ hoặc bất kỳ `bindingFailureConditions` nào là `True`, scheduler
xóa phần cấp phát và lập lịch lại Pod.
Quản trị viên cluster có thể cấu hình khoảng thời gian chờ này bằng cách chỉnh sửa file cấu hình kube-scheduler.

Dưới đây là ví dụ cấu hình timeout này trong `KubeSchedulerConfiguration`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  pluginConfig:
  - name: DynamicResources
    args:
      apiVersion: kubescheduler.config.k8s.io/v1
      kind: DynamicResourcesArgs
      bindingTimeout: 60s
```

#### Ví dụ (Example) {#device-binding-conditions-example}

Đây là ví dụ về một ResourceSlice mà bạn có thể thấy trong một cluster đang dùng một driver DRA, và driver đó hỗ trợ binding conditions:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: gpu-slice-1
spec:
  driver: dra.example.com
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
      - key: accelerator-type
        operator: In
        values:
        - "high-performance"
  pool:
    name: gpu-pool
    generation: 1
    resourceSliceCount: 1
  devices:
    - name: gpu-1
      attributes:
        vendor:
          string: "example"
        model:
          string: "example-gpu"
      bindsToNode: true
      bindingConditions:
        - dra.example.com/is-prepared
      bindingFailureConditions:
        - dra.example.com/preparing-failed
```

ResourceSlice ví dụ này có các đặc tính sau:

- ResourceSlice nhắm đến các node có nhãn `accelerator-type=high-performance`,
để scheduler chỉ dùng một tập cụ thể các node đủ điều kiện.
- Scheduler chọn một node từ nhóm đã chọn (ví dụ, `node-3`) và đặt
trường `status.allocation.nodeSelector` trong ResourceClaim thành tên node đó.
- Binding condition `dra.example.com/is-prepared` biểu thị rằng thiết bị `gpu-1`
phải được chuẩn bị (condition `is-prepared` có status là `True`) trước khi binding.
- Nếu việc chuẩn bị thiết bị `gpu-1` thất bại (condition `preparing-failed` có status là `True`), scheduler hủy binding.
- Scheduler chờ tối đa 600 giây (mặc định) để thiết bị trở nên sẵn sàng.
- Các controller bên ngoài có thể dùng node selector trong ResourceClaim để thực hiện
thiết lập đặc thù theo node trên node đã chọn.

Điều kiện gắn kết thiết bị là một *tính năng beta* và được bật mặc định, được kiểm soát bởi
[feature gate `DRADeviceBindingConditions`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRADeviceBindingConditions)
trong kube-apiserver và kube-scheduler.

### Tài nguyên node allocatable (Node allocatable resources) {#node-allocatable-resources}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Các thiết bị do DRA quản lý có thể có một dấu chân tài nguyên (footprint) bên dưới bao gồm các tài nguyên
node-allocatable, chẳng hạn `cpu`, `memory`, `hugepages`, hoặc `ephemeral-storage`.
Tính năng này tích hợp các yêu cầu dựa trên DRA đó vào cơ chế hạch toán chuẩn của
scheduler cùng với các yêu cầu thông thường trong `spec` của Pod cho những tài nguyên này.

Người dùng (tác giả PodSpec) có thể dùng kết hợp tài nguyên mức Pod, tài nguyên mức container,
và các resource claim có tài nguyên node-allocatable liên quan. Những thiết bị này có thể đại diện
trực tiếp cho các tài nguyên như CPU hoặc bộ nhớ, hoặc có thể là bộ tăng tốc, card mạng,
hoặc các thiết bị khác cần một phần tài nguyên của máy chủ (host) khi được cấp phát. Driver DRA sẽ
điền thông tin vào ResourceSlice cho scheduler biết cách tính toán các
tài nguyên node-allocatable khi thiết bị được cấp phát cho một ResourceClaim.
Tác giả PodSpec không cần tự thực hiện phép tính đó.

Khi viết một PodSpec dùng claim cho các loại thiết bị này, có một vài điều cần lưu ý:

*   Khi dùng tài nguyên mức Pod, tổng tất cả tài nguyên của container và claim
    không được vượt quá tài nguyên mức Pod; nếu không, Pod sẽ không được lập lịch.
*   Tổng yêu cầu tài nguyên của một container là tổng của tài nguyên mức container
    và mọi tài nguyên node-allocatable từ các resource claim liên quan của nó.
*   Các claim tiêu thụ tài nguyên node-allocatable không thể được chia sẻ giữa các Pod.

#### Chi tiết cho tác giả driver DRA (Details for DRA Driver Authors)

Các driver DRA khai báo dấu chân tài nguyên node-allocatable này bằng
trường `nodeAllocatableResourceMappings` trên các thiết bị trong một ResourceSlice.
Ánh xạ (mapping) này quy đổi thiết bị DRA hoặc dung lượng được yêu cầu thành các tài nguyên
chuẩn được theo dõi trong `status.allocatable` của node (lưu ý rằng extended
resource không được hỗ trợ trong ánh xạ này). Điều này hữu ích cho cả các driver trực tiếp
cung cấp tài nguyên gốc (như driver DRA cho CPU hoặc bộ nhớ) lẫn các thiết bị
cần các phụ thuộc phụ trợ trên node (như một bộ tăng tốc cần bộ nhớ của máy chủ).

Ánh xạ này định nghĩa phép quy đổi từ thiết bị DRA hoặc đơn vị dung lượng được yêu cầu
sang số lượng tương ứng của tài nguyên node-allocatable. Scheduler
tính toán số lượng chính xác bằng cách:

*   **Chia tỉ lệ theo thiết bị (Device-based scaling):** Nếu `capacityKey` không được đặt,
    `allocationMultiplier` nhân với số lượng thiết bị được cấp phát cho claim.
    `allocationMultiplier` mặc định là 1 nếu không được chỉ định.
*   **Chia tỉ lệ theo dung lượng (Capacity-based scaling):** Nếu `capacityKey` được đặt, nó tham chiếu đến một
    tên capacity được định nghĩa trong map `capacity` của thiết bị. Scheduler tra cứu
    lượng capacity đó mà claim tiêu thụ và nhân nó với
    `allocationMultiplier`.

##### Ví dụ: Driver DRA cho CPU — chia tỉ lệ theo dung lượng (Example: CPU DRA Driver (Capacity-based scaling))

Đây là ví dụ trong đó một driver DRA cho CPU cung cấp một socket CPU dưới dạng một pool gồm 128
CPU bằng [dung lượng tiêu thụ được của DRA](#consumable-capacity). `capacityKey` liên kết capacity
`cpu.example.com/cpu` được tiêu thụ trực tiếp tới tài nguyên allocatable `cpu`
chuẩn của node:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: my-node-cpus
spec:
  driver: cpu.example.com
  nodeName: my-node
  pool:
    name: socket-cpus
    generation: 1
    resourceSliceCount: 1
  devices:
  - name: socket0cpus
    allowMultipleAllocations: true
    capacity:
      "cpu.example.com/cpu": "128"
    nodeAllocatableResourceMappings:
      cpu:
        capacityKey: "cpu.example.com/cpu"
        # allocationMultiplier mặc định là 1 nếu bỏ qua
  - name: socket1cpus
    allowMultipleAllocations: true
    capacity:
      "cpu.example.com/cpu": "128"
    nodeAllocatableResourceMappings:
      cpu:
        capacityKey: "cpu.example.com/cpu"
        # allocationMultiplier mặc định là 1 nếu bỏ qua
```

##### Ví dụ: Bộ tăng tốc với tài nguyên phụ trợ — chia tỉ lệ theo thiết bị (Example: Accelerator with Auxiliary Resources (Device-based scaling))

Đây là ví dụ về một resource slice trong đó một bộ tăng tốc yêu cầu thêm
8Gi bộ nhớ cho mỗi thiết bị để hoạt động:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: my-node-xpus
spec:
  driver: xpu.example.com
  nodeName: my-node
  pool:
    name: xpu-pool
    generation: 1
    resourceSliceCount: 1
  devices:
  - name: xpu-model-x-001
    attributes:
      example.com/model:
        string: "model-x"
    nodeAllocatableResourceMappings:
      memory:
        allocationMultiplier: "8Gi"
```

Sau khi một Pod được bind thành công vào node, số lượng chính xác của
các tài nguyên node-allocatable được cấp phát qua DRA được đưa vào trường
`status.nodeAllocatableResourceClaimStatuses` của Pod.

Tài nguyên node-allocatable là một tính năng alpha và được bật khi
[feature gate `DRANodeAllocatableResources`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#DRANodeAllocatableResources) được bật trong kube-apiserver,
kube-scheduler, và kubelet. Ở giai đoạn alpha, kubelet không tính đến
các tài nguyên này khi xác định lớp QoS, cấu hình cgroup, hoặc ra
quyết định trục xuất.

### Metadata thiết bị DRA trong container (DRA device metadata in containers) {#device-metadata}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Các driver DRA có thể cung cấp metadata thiết bị như các thuộc tính thiết bị (địa chỉ PCI bus
hoặc mdevUUID cho các thiết bị trung gian — mediated devices) hoặc cấu hình mạng trực tiếp
cho các container dưới dạng file JSON.
Điều này cho phép các ứng dụng bên trong container khám phá thông tin về các thiết bị
đã cấp phát mà không cần truy vấn API Kubernetes hay xây dựng các controller tùy chỉnh.

KEP-5304 định nghĩa một
[giao thức metadata thiết bị](#device-metadata-protocol) mà các driver phải
tuân theo để các ứng dụng bên trong container thấy được một bố cục nhất quán giữa các
driver và các cluster. Thư viện
[DRA kubelet plugin](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin)
triển khai giao thức này cho bạn; phần còn lại của mục này mô tả cách
sử dụng nó.

Metadata thiết bị tuân theo cùng quy tắc với quyền truy cập thiết bị: nó chỉ khả dụng bên trong
một container khi container đó yêu cầu thiết bị trong đặc tả container
của nó, và không khả dụng trong trường hợp khác. Về cách yêu cầu thiết bị DRA trong Pod và
container, xem
[Yêu cầu thiết bị trong workload bằng DRA](270-allocate-devices-dra-vi.md#request-devices-workloads).

#### Giao thức metadata thiết bị (Device metadata protocol) {#device-metadata-protocol}

Giao thức bao gồm bốn quy tắc:

1. **Đường dẫn file.** Các file metadata nằm bên trong container tại
   `/var/run/kubernetes.io/dra-device-attributes`. Với một
   ResourceClaim được tham chiếu trực tiếp, đường dẫn là
   `resourceclaims/<claimName>/<requestName>/<driverName>-metadata.json`; với một
   claim được tạo từ ResourceClaimTemplate, đường dẫn là
   `resourceclaimtemplates/<podClaimName>/<requestName>/<driverName>-metadata.json`
   (trong đó `podClaimName` là `pod.spec.resourceClaims[].name`).

   Trong trường hợp request của ResourceClaim dùng tính năng
   [danh sách ưu tiên](#prioritized-list), chỉ tên request cấp cao nhất
   được dùng cho đoạn `<requestName>` trong đường dẫn file (tức là
   phần `/<subrequest>` bị lược bỏ). Bên trong
   file JSON, trường `requests[].name` mang tham chiếu đầy đủ
   `<request>/<subrequest>` (ví dụ, `gpu/high-memory`) để
   bên sử dụng có thể xác định phương án thay thế nào đã được cấp phát.

   Các hằng số đường dẫn được định nghĩa trong
   [`k8s.io/dynamic-resource-allocation/api/metadata`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/api/metadata).

1. **JSON API.** Mỗi file là một luồng (stream) gồm một hoặc nhiều đối tượng
   [`DeviceMetadata`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/api/metadata/v1alpha1#DeviceMetadata)
   được tuần tự hóa thành JSON có phiên bản với `apiVersion` và `kind`, theo
   các quy ước API của Kubernetes. Cùng một metadata được mã hóa một lần cho mỗi phiên bản
   API được hỗ trợ (phiên bản mới nhất trước). Tất cả đối tượng trong luồng đều tương đương
   về ngữ nghĩa; bên sử dụng nên dùng đối tượng đầu tiên mà chúng giải mã được.

1. **Generation.** Khi một driver cập nhật file metadata, trường
   `metadata.generation` nhúng bên trong phải tăng để bên sử dụng có thể phát hiện thay đổi.

1. **Cách đưa vào container.** Các file thường được đưa vào qua
   bind-mount CDI, nhưng các cơ chế khác
   cũng được phép miễn là file xuất hiện đúng đường dẫn và
   ở chế độ chỉ đọc (read-only) bên trong container.

#### Cách metadata thiết bị hoạt động (How device metadata works) {#device-metadata-how-it-works}

Metadata thiết bị là một tính năng phía driver, không yêu cầu bất kỳ thay đổi API
Kubernetes hay feature gate nào. Dùng thư viện DRA kubelet plugin là cách phổ biến
để triển khai một driver, nhưng driver cũng có thể được xây dựng theo những cách khác.
Các driver dùng kubelet plugin bật tính năng này bằng cách truyền các
[tùy chọn](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin#Option)
`EnableDeviceMetadata` và `MetadataVersions`
khi khởi động plugin. `MetadataVersions` chỉ định các phiên bản API nào được
tuần tự hóa vào file metadata và phải được driver đặt một cách tường minh.
Kiểm tra tài liệu của driver DRA của bạn để biết liệu metadata thiết bị có được
hỗ trợ hay không và cách bật nó.

Khi metadata thiết bị được bật, driver sinh các file metadata và các đặc tả
bind-mount CDI trong lúc chuẩn bị các thiết bị đã cấp phát cho Pod,
trước khi các container sử dụng chúng khởi động. Metadata xuất hiện bên trong container tại
các đường dẫn quy ước như [định nghĩa ở trên](#device-metadata-protocol).

Khi một request duy nhất cấp phát thiết bị từ nhiều driver DRA, mỗi driver
ghi file metadata của riêng nó. Các container liệt kê các file `*-metadata.json` trong
thư mục của request để khám phá tất cả các thiết bị.

Gói Go
[`k8s.io/dynamic-resource-allocation/devicemetadata`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/devicemetadata)
cung cấp các tiện ích để các ứng dụng bên trong container đọc và giải mã
các file metadata này.

#### Lược đồ metadata (Metadata schema) {#device-metadata-schema}

Mỗi file metadata tuân theo API
[`DeviceMetadata`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/api/metadata/v1alpha1#DeviceMetadata)
(`metadata.resource.k8s.io/v1alpha1`).
Ví dụ sau cho thấy một file metadata cho một thiết bị GPU được cấp phát qua
một ResourceClaimTemplate:

```json
{
  "kind": "DeviceMetadata",
  "apiVersion": "metadata.resource.k8s.io/v1alpha1",
  "metadata": {
    "name": "pod0-gpu-2kqrd",
    "namespace": "gpu-test1",
    "uid": "c7e7b22e-239b-4498-b27c-7f1344481e14",
    "generation": 1
  },
  "podClaimName": "gpu",
  "requests": [
    {
      "name": "gpu",
      "devices": [
        {
          "driver": "gpu.example.com",
          "pool": "worker-0",
          "name": "gpu-0",
          "attributes": {
            "driverVersion": {
              "version": "1.0.0"
            },
            "index": {
              "int": 0
            },
            "model": {
              "string": "LATEST-GPU-MODEL"
            },
            "uuid": {
              "string": "gpu-18db0e85-99e9-c746-8531-ffeb86328b39"
            }
          }
        }
      ]
    }
  ]
}
```

#### Metadata tức thời và trì hoãn (Immediate and deferred metadata) {#device-metadata-lifecycle}

Driver cung cấp metadata theo một trong hai cách:

Tức thời (Immediate)
: Driver điền metadata trong lúc chuẩn bị claim trên
  node và ghi file metadata trước khi container khởi động. Đây là cách
  điển hình cho các driver GPU, khi thông tin thiết bị đã được biết tại thời điểm chuẩn bị.

Trì hoãn (Deferred)
: Trong một số trường hợp, ví dụ driver mạng, thông tin thiết bị
  không có sẵn tại thời điểm cấp phát thiết bị mà chỉ có sau khi
  pod sandbox được tạo. Trong những trường hợp đó, driver tạo CDI mount với
  một file metadata rỗng và ghi metadata thực sự sau đó thông qua một NRI hook
  chạy trước khi container khởi động. Điều này đảm bảo ứng dụng không bao giờ thấy một
  file bị thiếu hoặc ghi dở dang. Mỗi lần cập nhật phải tăng
  `metadata.generation` để bên sử dụng có thể phát hiện thay đổi. API `MetadataUpdater`
  trong thư viện DRA kubelet plugin tự động xử lý việc quản lý generation
  cho các tác giả driver.

Trong cả hai trường hợp, metadata vẫn khả dụng cho mỗi container sử dụng trong suốt
thời gian tồn tại của container đó. Các file metadata được dọn dẹp sau khi tất cả container
trong Pod đã kết thúc.

Để tìm hiểu cách dùng metadata thiết bị trong workload của bạn, xem
[Truy cập metadata thiết bị DRA](269-access-dra-device-metadata-vi.md).

#### Driver tùy chỉnh (Custom drivers) {#device-metadata-custom-drivers}

Các driver tùy chỉnh, tự xây dựng mà không dùng thư viện DRA kubelet plugin
phải tự triển khai [giao thức metadata thiết bị](#device-metadata-protocol).
Điều đó nghĩa là ghi JSON `DeviceMetadata` tại đúng các đường dẫn file,
tăng `metadata.generation` trong mỗi lần cập nhật, và đưa các file
vào bên trong container ở chế độ chỉ đọc thông qua CDI hoặc một cơ chế tương đương.

### Thuộc tính kiểu danh sách (List type attributes) {#list-type-attributes}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Tính năng này cải tiến API ResourceSlice, cho phép các driver DRA chỉ định giá trị danh sách cho các thuộc tính thiết bị thay vì chỉ các giá trị vô hướng (scalar).
Điều này hữu ích để mô hình hóa các cấu trúc liên kết (topology) nội bộ phức tạp hơn của node, ví dụ khi một CPU kề cận nhiều PCIe root.

Với các tác giả ResourceClaim (người dùng cuối), điều này nghĩa là `matchAttribute` và `distinctAttribute` hoạt động tốt hơn cho các trường hợp này.

- `matchAttribute` — hai thuộc tính phải có *phần giao danh sách khác rỗng*, thay vì phải giống hệt nhau (các giá trị vô hướng được coi như danh sách một phần tử).
  Điều này đơn giản nghĩa là nếu một driver công bố một giá trị đơn cho, chẳng hạn, PCIe root, và một driver khác công bố một danh sách, thì ràng buộc được thỏa mãn miễn là
  giá trị đơn đó xuất hiện đâu đó trong danh sách.
- `distinctAttribute` — các giá trị thuộc tính phải *rời nhau từng đôi một* (không có giá trị nào được chia sẻ giữa bất kỳ hai thiết bị nào)

Để giúp các tác giả ResourceClaim dùng những thuộc tính có thể là danh sách bên trong biểu thức CEL, tính năng này cũng giới thiệu hàm CEL `includes()`.

```
# Thuộc tính vô hướng (tương thích ngược)
# giả sử: device.attributes["dra.example.com"].model = "model-a"
device.attributes["dra.example.com"].model.includes("model-a")  # true
device.attributes["dra.example.com"].model.includes("model-b")  # false

# Thuộc tính kiểu danh sách (yêu cầu DRAListTypeAttributes)
# giả sử: device.attributes["dra.example.com"].supported-models= ["model-a", "model-b"]
device.attributes["dra.example.com"].supported-models.includes("model-a")  # true
device.attributes["dra.example.com"].supported-models.includes("model-c")  # false
```

#### Chi tiết cho tác giả driver DRA (Details for DRA Driver Authors)

Theo mặc định, mỗi `DeviceAttribute` chứa đúng một giá trị vô hướng: một boolean, một số nguyên,
một chuỗi, hoặc một chuỗi phiên bản ngữ nghĩa (semantic version). Feature gate `DRAListTypeAttributes` mở rộng
`DeviceAttribute` với bốn trường kiểu danh sách, cho phép một thiết bị công bố nhiều
giá trị cho một thuộc tính duy nhất:

- **`bools`** — danh sách các giá trị boolean
- **`ints`** — danh sách các giá trị số nguyên 64-bit
- **`strings`** — danh sách các chuỗi (mỗi chuỗi tối đa 64 ký tự)
- **`versions`** — danh sách các chuỗi phiên bản ngữ nghĩa theo đặc tả semver.org 2.0.0
  (mỗi chuỗi tối đa 64 ký tự)

Tổng số giá trị thuộc tính riêng lẻ trên mỗi thiết bị (các trường vô hướng cộng toàn bộ
phần tử danh sách) bị giới hạn ở mức **48**. Khi bất kỳ thiết bị nào trong một ResourceSlice dùng tính năng này hoặc các tính năng nâng cao khác như taint,
ResourceSlice đó sẽ bị giới hạn tối đa **64** thiết bị.

Đây là ví dụ về một thiết bị công bố nhiều model được hỗ trợ bằng một thuộc tính
chuỗi kiểu danh sách:

```yaml
kind: ResourceSlice
apiVersion: resource.k8s.io/v1
metadata:
  name: example-resourceslice
spec:
  nodeName: worker-1
  pool:
    name: pool
    generation: 1
    resourceSliceCount: 1
  driver: dra.example.com
  devices:
  - name: gpu-0
    attributes:
      dra.example.com/supported-models:
        strings:
        - model-a
        - model-b
```

Thuộc tính kiểu danh sách là một *tính năng alpha* và chỉ được bật khi
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `DRAListTypeAttributes`
được bật trong kube-apiserver và kube-scheduler.

## Tiếp theo (What's next)

- [Thiết lập DRA trong cluster](271-set-up-dra-cluster-vi.md)
- [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md)
- [Truy cập metadata thiết bị DRA](269-access-dra-device-metadata-vi.md)
- Để biết thêm thông tin về thiết kế, xem KEP
  [Dynamic Resource Allocation with Structured Parameters](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Trong bốn kind API của DRA, kind nào do driver thiết bị tạo ra để công bố phần cứng, và
   kind nào do người vận hành workload tạo ra để xin dùng phần cứng đó?
2. DRA khác device plugin truyền thống ở những điểm nào? Vì sao viết `example.com/gpu: 2` vào
   `resources.limits` của container **không** phải là cách làm việc của DRA?
3. Cluster lab của bạn không có GPU và không cài driver DRA nào. Nếu bạn vẫn tạo một Pod tham
   chiếu ResourceClaim, Pod đó ra sao? Và nếu sau này cluster có GPU, một Pod ưu tiên cao có
   giành được GPU từ một Pod ưu tiên thấp đang giữ nó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Driver tạo **ResourceSlice** — mỗi ResourceSlice đại diện cho một hoặc nhiều thiết bị trong
   một pool, kèm thuộc tính, dung lượng và danh sách node truy cập được; driver cũng chạy một
   controller đối chiếu và **ghi đè mọi thay đổi thủ công** lên ResourceSlice. Người vận hành
   workload tạo **ResourceClaim** hoặc **ResourceClaimTemplate**. **DeviceClass** ở giữa: do
   admin *hoặc* driver tạo, định nghĩa danh mục thiết bị và cách chọn thuộc tính.
2. Bài liệt kê đúng ba thiếu sót của device plugin: nó **yêu cầu khai báo thiết bị theo từng
   container**, **không hỗ trợ chia sẻ thiết bị**, và **không lọc thiết bị dựa trên biểu
   thức**. DRA lật cả ba: lọc chi tiết bằng CEL theo thuộc tính thiết bị, chia sẻ một claim
   cho nhiều container hoặc nhiều Pod, và **Pod không chỉ định số lượng thiết bị** — Pod chỉ
   tham chiếu một ResourceClaim, còn cấu hình thiết bị nằm trong claim đó. Đếm số trong
   `resources.limits` chính là mô hình device plugin, tức là thứ DRA thay thế. (Ngoại lệ duy
   nhất — *Cấp phát extended resource bằng DRA* — là một tính năng beta riêng dựng cầu nối
   ngược lại cho người dùng cũ, không phải đường đi mặc định.)
3. Pod sẽ **không được lập lịch**. Ở bước *Lọc ResourceSlice*, Kubernetes phải tìm được một
   ResourceSlice có tài nguyên chưa cấp phát khớp yêu cầu của claim; không driver thì không
   ResourceSlice, nên không có gì để khớp. (Nếu ResourceClaim được tham chiếu trực tiếp mà
   không tồn tại trong đúng namespace của Pod, Pod cũng không được lập lịch — hệt như
   PersistentVolumeClaim.) Câu sau: **không**. Mục *Hạn chế* nói rõ scheduler không hỗ trợ
   preemption cho tài nguyên DRA; Pod ưu tiên cao nằm pending cho tới khi Pod đang giữ thiết
   bị kết thúc hoặc bị xóa thủ công.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
