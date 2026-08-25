# StatefulSets

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>
>
> StatefulSet chạy một nhóm Pod, và duy trì một định danh cố định (sticky identity) cho mỗi Pod trong số đó. Điều này hữu ích để quản lý các ứng dụng cần lưu trữ bền vững (persistent storage) hoặc một định danh mạng ổn định, duy nhất.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 4/14 ·
Kiểm chứng ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md).

Bài này đứng chân trên hai thứ bạn **chưa học**: lưu trữ bền vững và Service headless. Đó là
lý do lộ trình cố ý cắt đôi nó. Ở giai đoạn 4 bạn chỉ thực hành **định danh ổn định và thứ
tự khởi tạo** — hai thứ chạy được trên cluster baseline. Phần `volumeClaimTemplates` là
[nợ lab](labs/README.md#5-sổ-nợ-lab) trả ở **Lab 6a** (cần StorageClass và provisioner của
giai đoạn 6); phần Service quản trị headless là nợ trả ở **Lab 5a** (cần Service headless
của giai đoạn 5). Đừng cài provisioner sớm để "chạy thử cho đủ".

**Phải hiểu ở lần đọc này:**

- **Định danh cố định**: hostname theo mẫu `$(statefulset name)-$(ordinal)`, số thứ tự chạy
  từ 0 đến N-1, và định danh này **gắn chặt với Pod bất kể Pod được lập lịch lên node nào**
  (mục *Định danh của Pod* và *Chỉ số thứ tự*).
- **Bảo đảm thứ tự** với `podManagementPolicy` mặc định `OrderedReady`: tạo tuần tự 0→N-1 và
  Pod sau chỉ khởi chạy khi mọi Pod trước đã Running và Ready; kết thúc theo thứ tự ngược
  N-1→0 (mục *Các đảm bảo về triển khai và mở rộng*). `Parallel` bỏ việc chờ nhưng **vẫn giữ**
  bảo đảm về tính duy nhất và định danh.
- **RollingUpdate của StatefulSet đi ngược chiều**: từ ordinal lớn nhất xuống nhỏ nhất, từng
  Pod một, chờ Pod vừa cập nhật Running và Ready rồi mới sang Pod đứng trước. `partition` giữ
  mọi Pod có ordinal nhỏ hơn ở phiên bản cũ.
- Ba ràng buộc trong mục *Hạn chế* phải thuộc: StatefulSet **yêu cầu một Headless Service do
  bạn tự tạo**; xóa hoặc thu nhỏ StatefulSet **không** xóa volume; và không có bảo đảm nào về
  việc kết thúc Pod khi xóa StatefulSet — muốn êm thì thu nhỏ về 0 trước.
- Cái bẫy *Rollback bắt buộc*: template hỏng làm rollout dừng và chờ mãi, và **hoàn nguyên
  template thôi là chưa đủ** — bạn còn phải xóa tay các Pod đã chạy với cấu hình lỗi.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Template cho volume claim*, *Lưu trữ ổn định*, *Lưu giữ PersistentVolumeClaim* | baseline chưa có StorageClass, chưa học PV/PVC | giai đoạn 6 — [nợ lab](labs/README.md#5-sổ-nợ-lab) trả ở Lab 6a |
| *Định danh mạng ổn định* — bảng tên DNS, negative caching, CoreDNS | chưa học Service và DNS | giai đoạn 5 — [nợ lab](labs/README.md#5-sổ-nợ-lab) trả ở Lab 5a |
| *Số Pod không khả dụng tối đa* (`maxUnavailable`) | beta và tắt mặc định | không cần |
| *Lịch sử revision* — ControllerRevision, *Thực hành tốt*, *Giám sát* | cơ chế revision đã nắm qua Deployment | bài [63](63-deployment-vi.md) |
| *Số thứ tự bắt đầu* (`.spec.ordinals`) và *Nhãn chỉ số Pod* | tình huống hiếm, phục vụ định tuyến theo index | giai đoạn 5 |

---

StatefulSet là đối tượng API workload được dùng để quản lý các ứng dụng có trạng thái (stateful applications).

StatefulSet quản lý việc triển khai (deployment) và mở rộng (scaling) một tập các Pod, *đồng thời cung cấp các đảm bảo về thứ tự và tính duy nhất* của các Pod này.

Giống như một Deployment, StatefulSet quản lý các Pod dựa trên cùng một spec container giống hệt nhau. Khác với Deployment, StatefulSet duy trì một định danh cố định (sticky identity) cho mỗi Pod của nó. Các pod này được tạo từ cùng một spec, nhưng không thể hoán đổi cho nhau: mỗi Pod có một định danh bền vững được giữ nguyên qua bất kỳ lần lập lịch lại (rescheduling) nào.

Nếu bạn muốn dùng các volume lưu trữ để cung cấp tính bền vững cho workload của mình, bạn có thể dùng StatefulSet như một phần của giải pháp. Mặc dù từng Pod riêng lẻ trong StatefulSet vẫn có thể gặp sự cố, các định danh Pod bền vững giúp việc khớp các volume hiện có với các Pod mới thay thế cho những Pod đã hỏng trở nên dễ dàng hơn.

## Sử dụng StatefulSet (Using StatefulSets) {#using-statefulsets}

StatefulSet có giá trị đối với các ứng dụng yêu cầu một hoặc nhiều điều sau:

* Định danh mạng ổn định, duy nhất.
* Lưu trữ ổn định, bền vững.
* Triển khai và mở rộng có thứ tự, nhẹ nhàng (graceful).
* Cập nhật cuốn chiếu (rolling update) tự động, có thứ tự.

Trong các điều trên, "ổn định" đồng nghĩa với việc được duy trì qua các lần Pod được lập lịch (lại).
Nếu một ứng dụng không yêu cầu bất kỳ định danh ổn định nào hay việc triển khai, xóa,
hoặc mở rộng có thứ tự, bạn nên triển khai ứng dụng của mình bằng một đối tượng workload
cung cấp một tập các bản sao (replica) không trạng thái (stateless).
[Deployment](63-deployment-vi.md) hoặc
[ReplicaSet](64-replicaset-vi.md) có thể phù hợp hơn với các nhu cầu stateless của bạn.

## Hạn chế (Limitations) {#limitations}

* Lưu trữ cho một Pod nhất định phải được cấp phát (provision) bởi một
  [PersistentVolume Provisioner](98-dynamic-provisioning-vi.md)
  dựa trên _storage class_ được yêu cầu, hoặc được admin cấp phát sẵn từ trước.
* Việc xóa và/hoặc thu nhỏ (scale down) một StatefulSet sẽ _không_ xóa các volume gắn với
  StatefulSet đó. Điều này nhằm đảm bảo an toàn dữ liệu, vốn thường có giá trị hơn so với
  việc tự động dọn sạch toàn bộ các tài nguyên liên quan của StatefulSet.
* StatefulSet hiện yêu cầu một [Headless Service](82-service-vi.md#headless-services)
  chịu trách nhiệm về định danh mạng của các Pod. Bạn có trách nhiệm tạo Service này.
* StatefulSet không cung cấp bất kỳ đảm bảo nào về việc kết thúc (termination) các pod khi một
  StatefulSet bị xóa. Để đạt được việc kết thúc các pod trong StatefulSet một cách có thứ tự và
  nhẹ nhàng, có thể thu nhỏ StatefulSet xuống 0 trước khi xóa.
* Khi dùng [Rolling Update](#rolling-updates) với
  [chính sách quản lý Pod](#pod-management-policies) mặc định (`OrderedReady`),
  có thể rơi vào trạng thái hỏng đòi hỏi
  [can thiệp thủ công để sửa chữa](#forced-rollback).

## Các thành phần (Components) {#components}

Ví dụ dưới đây minh họa các thành phần của một StatefulSet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # phải khớp với .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # mặc định là 1
  minReadySeconds: 10 # mặc định là 0
  template:
    metadata:
      labels:
        app: nginx # phải khớp với .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.24
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

> **Ghi chú:**
> Ví dụ này dùng access mode `ReadWriteOnce` cho đơn giản. Với môi trường
> production, dự án Kubernetes khuyến nghị dùng access mode `ReadWriteOncePod`
> thay thế.

Trong ví dụ trên:

* Một Headless Service, tên là `nginx`, được dùng để kiểm soát miền mạng (network domain).
* StatefulSet, tên là `web`, có một Spec chỉ ra rằng 3 bản sao của container nginx sẽ được khởi chạy trong các Pod riêng biệt.
* `volumeClaimTemplates` sẽ cung cấp lưu trữ ổn định bằng các
  [PersistentVolume](92-persistent-volumes-vi.md) do một
  PersistentVolume Provisioner cấp phát.

Tên của một đối tượng StatefulSet phải là một
[nhãn DNS (DNS label)](17-names-vi.md#dns-label-names) hợp lệ.

### Pod Selector {#pod-selector}

Bạn phải đặt trường `.spec.selector` của một StatefulSet khớp với các nhãn trong
`.spec.template.metadata.labels` của nó. Việc không chỉ định một Pod Selector khớp sẽ dẫn đến
lỗi kiểm tra hợp lệ (validation error) khi tạo StatefulSet.

### Template cho volume claim (Volume Claim Templates) {#volume-claim-templates}

Bạn có thể đặt trường `.spec.volumeClaimTemplates` để tạo một
[PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims).
Điều này sẽ cung cấp lưu trữ ổn định cho StatefulSet nếu một trong hai điều kiện sau thỏa mãn:

* StorageClass được chỉ định cho volume claim được thiết lập để dùng
  [cấp phát động (dynamic provisioning)](98-dynamic-provisioning-vi.md).
* Cluster đã có sẵn một PersistentVolume với StorageClass đúng
  và đủ dung lượng lưu trữ khả dụng.

### Số giây sẵn sàng tối thiểu (Minimum ready seconds) {#minimum-ready-seconds}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

`.spec.minReadySeconds` là một trường tùy chọn chỉ định số giây tối thiểu mà một Pod mới
được tạo phải chạy và sẵn sàng, không có container nào của nó bị crash, để nó được xem là khả dụng (available).
Trường này được dùng để kiểm tra tiến độ của một lần rollout khi dùng chiến lược [Rolling Update](#rolling-updates).
Trường này mặc định là 0 (Pod sẽ được xem là khả dụng ngay khi nó sẵn sàng). Để tìm hiểu thêm về khi nào
một Pod được xem là sẵn sàng, xem [Container Probes](47-pod-lifecycle-vi.md#container-probes).

## Định danh của Pod (Pod Identity) {#pod-identity}

Các Pod của StatefulSet có một định danh duy nhất bao gồm một số thứ tự (ordinal), một
định danh mạng ổn định, và lưu trữ ổn định. Định danh này gắn chặt với Pod,
bất kể Pod được lập lịch (lại) lên node nào.

### Chỉ số thứ tự (Ordinal Index) {#ordinal-index}

Với một StatefulSet có N [replica](#replicas), mỗi Pod trong StatefulSet
sẽ được gán một số thứ tự nguyên, duy nhất trong toàn bộ Set. Mặc định,
các pod sẽ được gán các số thứ tự từ 0 đến N-1. StatefulSet controller
cũng sẽ thêm một nhãn pod chứa chỉ số này: `apps.kubernetes.io/pod-index`.

### Số thứ tự bắt đầu (Start ordinal) {#start-ordinal}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

`.spec.ordinals` là một trường tùy chọn cho phép bạn cấu hình các số thứ tự nguyên
được gán cho mỗi Pod. Giá trị mặc định là nil. Bên trong trường này, bạn có thể
cấu hình các tùy chọn sau:

* `.spec.ordinals.start`: Nếu trường `.spec.ordinals.start` được đặt, các Pod sẽ
  được gán các số thứ tự từ `.spec.ordinals.start` đến
  `.spec.ordinals.start + .spec.replicas - 1`.

### Định danh mạng ổn định (Stable Network ID) {#stable-network-id}

Mỗi Pod trong một StatefulSet lấy hostname của nó từ tên của StatefulSet
và số thứ tự của Pod. Mẫu cho hostname được tạo ra
là `$(statefulset name)-$(ordinal)`. Ví dụ ở trên sẽ tạo ba Pod
tên là `web-0,web-1,web-2`.
Một StatefulSet có thể dùng một [Headless Service](82-service-vi.md#headless-services)
để kiểm soát miền (domain) của các Pod của nó. Miền do Service này quản lý có dạng:
`$(service name).$(namespace).svc.cluster.local`, trong đó "cluster.local" là
miền của cluster.
Khi mỗi Pod được tạo, nó nhận một subdomain DNS tương ứng, có dạng:
`$(podname).$(governing service domain)`, trong đó Service quản lý (governing service) được xác định
bởi trường `serviceName` trên StatefulSet.

Tùy vào cách DNS được cấu hình trong cluster của bạn, có thể bạn không tra cứu được ngay
tên DNS của một Pod vừa mới chạy. Hành vi này có thể xảy ra khi các client khác trong
cluster đã gửi truy vấn cho hostname của Pod trước khi Pod được tạo.
Negative caching (điều bình thường trong DNS) nghĩa là kết quả của các lần tra cứu thất bại trước đó
được ghi nhớ và tái sử dụng, ngay cả sau khi Pod đã chạy, trong ít nhất vài giây.

Nếu bạn cần phát hiện các Pod ngay sau khi chúng được tạo, bạn có một vài lựa chọn:

* Truy vấn trực tiếp Kubernetes API (ví dụ, dùng watch) thay vì dựa vào việc tra cứu DNS.
* Giảm thời gian cache trong DNS provider của Kubernetes (thường điều này nghĩa là sửa
  config map của CoreDNS, hiện đang cache trong 30 giây).

Như đã đề cập trong phần [hạn chế](#limitations), bạn có trách nhiệm
tạo [Headless Service](82-service-vi.md#headless-services)
chịu trách nhiệm về định danh mạng của các pod.

Dưới đây là một số ví dụ về các lựa chọn cho Cluster Domain, tên Service,
tên StatefulSet, và cách chúng ảnh hưởng đến tên DNS của các Pod thuộc StatefulSet.

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                      | Pod Hostname |
| -------------- | ----------------- | --------------------- | ------------------------------- | -------------------------------------------- | ------------ |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

> **Ghi chú:**
> Cluster Domain sẽ được đặt là `cluster.local` trừ khi
> [được cấu hình khác đi](10-dns-pod-service-vi.md).

### Lưu trữ ổn định (Stable Storage) {#stable-storage}

Với mỗi mục VolumeClaimTemplate được định nghĩa trong một StatefulSet, mỗi Pod nhận một
PersistentVolumeClaim. Trong ví dụ nginx ở trên, mỗi Pod nhận một PersistentVolume duy nhất
với StorageClass là `my-storage-class` và 1 GiB dung lượng lưu trữ được cấp phát. Nếu không
chỉ định StorageClass, StorageClass mặc định sẽ được dùng. Khi một Pod được lập lịch (lại)
lên một node, các `volumeMounts` của nó sẽ mount các PersistentVolume gắn với các
PersistentVolume Claim của nó. Lưu ý rằng, các PersistentVolume gắn với các
PersistentVolume Claim của các Pod không bị xóa khi các Pod, hay StatefulSet bị xóa.
Việc này phải được thực hiện thủ công.

### Nhãn tên Pod (Pod Name Label) {#pod-name-label}

Khi StatefulSet controller tạo một Pod,
nó thêm một nhãn, `statefulset.kubernetes.io/pod-name`, được đặt bằng tên của
Pod. Nhãn này cho phép bạn gắn một Service vào một Pod cụ thể trong
StatefulSet.

### Nhãn chỉ số Pod (Pod index label) {#pod-index-label}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

Khi StatefulSet controller tạo một Pod,
Pod mới được gắn nhãn `apps.kubernetes.io/pod-index`. Giá trị của nhãn này là chỉ số thứ tự của
Pod. Nhãn này cho phép bạn định tuyến lưu lượng (traffic) tới một chỉ số pod cụ thể, lọc log/metrics
bằng nhãn chỉ số pod, và nhiều việc khác. Lưu ý rằng feature gate `PodIndexLabel` được bật và khóa
mặc định cho tính năng này; để tắt nó, người dùng sẽ phải dùng server emulated version v1.31.

## Các đảm bảo về triển khai và mở rộng (Deployment and Scaling Guarantees) {#deployment-and-scaling-guarantees}

* Với một StatefulSet có N replica, khi các Pod đang được triển khai, chúng được tạo tuần tự, theo thứ tự từ {0..N-1}.
* Khi các Pod đang bị xóa, chúng bị kết thúc theo thứ tự ngược lại, từ {N-1..0}.
* Trước khi một thao tác mở rộng (scaling) được áp dụng cho một Pod, tất cả các Pod đứng trước nó phải ở trạng thái Running và Ready. Nếu [`.spec.minReadySeconds`](#minimum-ready-seconds) được đặt, các Pod đứng trước phải khả dụng (Ready trong ít nhất `minReadySeconds`).
* Trước khi một Pod bị kết thúc, tất cả các Pod đứng sau nó phải được tắt (shutdown) hoàn toàn.

StatefulSet không nên chỉ định `pod.Spec.TerminationGracePeriodSeconds` bằng 0. Cách làm này
không an toàn và rất không được khuyến khích. Để được giải thích thêm, vui lòng tham khảo
[force delete Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md).

Khi ví dụ nginx ở trên được tạo, ba Pod sẽ được triển khai theo thứ tự
web-0, web-1, web-2. web-1 sẽ không được triển khai trước khi web-0
[Running và Ready](47-pod-lifecycle-vi.md), và web-2 sẽ không được triển khai cho đến khi
web-1 Running và Ready. Nếu web-0 gặp sự cố, sau khi web-1 đã Running và Ready, nhưng trước khi
web-2 được khởi chạy, thì web-2 sẽ không được khởi chạy cho đến khi web-0 được khởi chạy lại
thành công và trở thành Running và Ready.

Nếu người dùng scale ví dụ đã triển khai bằng cách patch StatefulSet sao cho
`replicas=1`, web-2 sẽ bị kết thúc trước tiên. web-1 sẽ không bị kết thúc cho đến khi web-2
được tắt hoàn toàn và bị xóa. Nếu web-0 gặp sự cố sau khi web-2 đã bị kết thúc và
tắt hoàn toàn, nhưng trước khi web-1 bị kết thúc, thì web-1 sẽ không bị kết thúc
cho đến khi web-0 Running và Ready.

### Các chính sách quản lý Pod (Pod Management Policies) {#pod-management-policies}

StatefulSet cho phép bạn nới lỏng các đảm bảo về thứ tự của nó trong khi
vẫn giữ các đảm bảo về tính duy nhất và định danh, thông qua trường `.spec.podManagementPolicy`.

#### Quản lý Pod kiểu OrderedReady (OrderedReady Pod Management) {#orderedready-pod-management}

Quản lý pod kiểu `OrderedReady` là mặc định cho StatefulSet. Nó thực hiện hành vi
được mô tả trong [Các đảm bảo về triển khai và mở rộng](#deployment-and-scaling-guarantees).

#### Quản lý Pod kiểu Parallel (Parallel Pod Management) {#parallel-pod-management}

Quản lý pod kiểu `Parallel` chỉ thị cho StatefulSet controller khởi chạy hoặc
kết thúc tất cả các Pod song song, và không chờ các Pod trở thành Running
và Ready hay kết thúc hoàn toàn trước khi khởi chạy hoặc kết thúc một
Pod khác.

Với các thao tác mở rộng, điều này nghĩa là tất cả các Pod được tạo hoặc kết thúc đồng thời.

Với các lần rolling update khi [`.spec.updateStrategy.rollingUpdate.maxUnavailable`](#maximum-unavailable-pods)
lớn hơn 1, StatefulSet controller kết thúc và tạo tối đa `maxUnavailable` Pod
cùng lúc (còn được gọi là "bursting"). Điều này có thể tăng tốc các lần cập nhật nhưng có thể khiến các Pod trở nên sẵn sàng không theo thứ tự, điều có thể không phù hợp với các ứng dụng đòi hỏi thứ tự nghiêm ngặt.

## Các chiến lược cập nhật (Update strategies) {#update-strategies}

Trường `.spec.updateStrategy` của một StatefulSet cho phép bạn cấu hình
và tắt tính năng rolling update tự động cho các container, nhãn, resource request/limit, và
annotation của các Pod trong một StatefulSet. Có hai giá trị khả dĩ:

`OnDelete`
: Khi `.spec.updateStrategy.type` của một StatefulSet được đặt là `OnDelete`,
  StatefulSet controller sẽ không tự động cập nhật các Pod trong một
  StatefulSet. Người dùng phải xóa các Pod một cách thủ công để khiến controller
  tạo các Pod mới phản ánh những thay đổi đã được thực hiện với `.spec.template` của StatefulSet.

`RollingUpdate`
: Chiến lược cập nhật `RollingUpdate` thực hiện việc cập nhật cuốn chiếu tự động cho các Pod trong một
  StatefulSet. Đây là chiến lược cập nhật mặc định.

## Cập nhật cuốn chiếu (Rolling Updates) {#rolling-updates}

Khi `.spec.updateStrategy.type` của một StatefulSet được đặt là `RollingUpdate`,
StatefulSet controller sẽ xóa và tạo lại từng Pod trong StatefulSet. Nó sẽ tiến hành
theo cùng thứ tự với việc kết thúc Pod (từ số thứ tự lớn nhất đến nhỏ nhất), cập nhật
lần lượt từng Pod một.

Control plane của Kubernetes chờ cho đến khi một Pod đã cập nhật trở thành Running và Ready
trước khi cập nhật Pod đứng trước nó. Nếu bạn đã đặt `.spec.minReadySeconds` (xem
[Số giây sẵn sàng tối thiểu](#minimum-ready-seconds)), control plane còn chờ thêm khoảng
thời gian đó sau khi Pod trở nên ready, rồi mới tiếp tục.

### Cập nhật cuốn chiếu theo phân vùng (Partitioned rolling updates) {#partitions}

Chiến lược cập nhật `RollingUpdate` có thể được phân vùng (partition), bằng cách chỉ định
`.spec.updateStrategy.rollingUpdate.partition`. Nếu một partition được chỉ định, tất cả các Pod có
số thứ tự lớn hơn hoặc bằng partition sẽ được cập nhật khi `.spec.template` của StatefulSet
được cập nhật. Tất cả các Pod có số thứ tự nhỏ hơn partition sẽ không
được cập nhật, và, ngay cả khi chúng bị xóa, chúng sẽ được tạo lại ở phiên bản trước đó. Nếu
`.spec.updateStrategy.rollingUpdate.partition` của một StatefulSet lớn hơn `.spec.replicas` của nó,
các cập nhật lên `.spec.template` của nó sẽ không được lan truyền tới các Pod của nó.
Trong hầu hết các trường hợp bạn sẽ không cần dùng partition, nhưng chúng hữu ích nếu bạn muốn
dàn dựng (stage) một bản cập nhật, triển khai một bản canary, hoặc thực hiện một đợt triển khai theo từng giai đoạn (phased roll out).

### Số Pod không khả dụng tối đa (Maximum unavailable Pods) {#maximum-unavailable-pods}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Bạn có thể kiểm soát số lượng Pod tối đa có thể không khả dụng trong quá trình cập nhật
bằng cách chỉ định trường `.spec.updateStrategy.rollingUpdate.maxUnavailable`.
Giá trị có thể là một số tuyệt đối (ví dụ, `5`) hoặc một tỷ lệ phần trăm của số Pod
mong muốn (ví dụ, `10%`). Số tuyệt đối được tính từ giá trị phần trăm
bằng cách làm tròn lên. Trường này không thể bằng 0. Giá trị mặc định là 1.

Trường này áp dụng cho tất cả các Pod trong khoảng từ `0` đến `replicas - 1`. Nếu có bất kỳ
Pod không khả dụng nào trong khoảng từ `0` đến `replicas - 1`, nó sẽ được tính vào
`maxUnavailable`.

> **Ghi chú:**
> Trường `maxUnavailable` đang ở giai đoạn Beta và bị tắt mặc định.

### Rollback bắt buộc (Forced rollback) {#forced-rollback}

Khi dùng [Rolling Update](#rolling-updates) với
[chính sách quản lý Pod](#pod-management-policies) mặc định (`OrderedReady`),
có thể rơi vào trạng thái hỏng đòi hỏi can thiệp thủ công để sửa chữa.

Nếu bạn cập nhật Pod template sang một cấu hình không bao giờ trở thành Running và
Ready (ví dụ, do một binary lỗi hoặc lỗi cấu hình ở tầng ứng dụng),
StatefulSet sẽ dừng việc rollout và chờ.

Trong trạng thái này, việc hoàn nguyên (revert) Pod template về cấu hình tốt là chưa đủ.
Do một [vấn đề đã biết](https://github.com/kubernetes/kubernetes/issues/67250),
StatefulSet sẽ tiếp tục chờ Pod hỏng trở thành Ready
(điều không bao giờ xảy ra) trước khi nó thử hoàn nguyên Pod đó về cấu hình
hoạt động được.

Sau khi hoàn nguyên template, bạn cũng phải xóa mọi Pod mà StatefulSet đã
thử chạy với cấu hình lỗi.
StatefulSet sau đó sẽ bắt đầu tạo lại các Pod bằng template đã hoàn nguyên.

## Lịch sử revision (Revision history) {#revision-history}

ControllerRevision là một tài nguyên API của Kubernetes được các controller, chẳng hạn như StatefulSet controller, dùng để theo dõi các thay đổi cấu hình trong quá khứ.

StatefulSet dùng các ControllerRevision để duy trì một lịch sử revision, cho phép rollback và theo dõi phiên bản.

### Cách StatefulSet theo dõi thay đổi bằng ControllerRevision (How StatefulSets track changes using ControllerRevisions) {#how-statefulsets-track-changes-using-controllerrevisions}

Khi bạn cập nhật Pod template của một StatefulSet (`spec.template`), StatefulSet controller sẽ:

1. Chuẩn bị một đối tượng ControllerRevision mới
2. Lưu một snapshot của Pod template và metadata
3. Gán một số revision tăng dần

#### Các thuộc tính chính (Key Properties) {#key-properties}

Xem [ControllerRevision](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1/) để tìm hiểu thêm về các thuộc tính chính và các chi tiết khác.

---

### Quản lý lịch sử revision (Managing Revision History) {#managing-revision-history}

Kiểm soát số revision được giữ lại bằng `.spec.revisionHistoryLimit`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: webapp
spec:
  revisionHistoryLimit: 5  # Giữ 5 revision gần nhất
  # ... các trường spec khác ...
```

* **Mặc định**: giữ lại 10 revision nếu không chỉ định
* **Dọn dẹp**: Các revision cũ nhất bị thu gom rác (garbage-collect) khi vượt quá giới hạn

#### Thực hiện rollback (Performing Rollbacks) {#performing-rollbacks}

Bạn có thể hoàn nguyên về một cấu hình trước đó bằng:

```bash
# Xem lịch sử revision
kubectl rollout history statefulset/webapp

# Rollback về một revision cụ thể
kubectl rollout undo statefulset/webapp --to-revision=3
```

Việc này sẽ:

* Áp dụng Pod template từ revision 3
* Tạo một ControllerRevision mới với số revision được cập nhật

#### Kiểm tra các ControllerRevision (Inspecting ControllerRevisions) {#inspecting-controllerrevisions}

Để xem các ControllerRevision liên quan:

```bash
# Liệt kê tất cả các revision của StatefulSet
kubectl get controllerrevisions -l app.kubernetes.io/name=webapp

# Xem cấu hình chi tiết của một revision cụ thể
kubectl get controllerrevision/webapp-3 -o yaml
```

#### Các thực hành tốt (Best Practices) {#best-practices}

##### Chính sách lưu giữ (Retention Policy) {#retention-policy}

- Đặt `revisionHistoryLimit` trong khoảng **5–10** cho hầu hết các workload.
- Chỉ tăng lên nếu cần **lịch sử rollback sâu**.

##### Giám sát (Monitoring) {#monitoring}

* Kiểm tra các revision định kỳ bằng:

  ```bash
  kubectl get controllerrevisions
  ```

- Cảnh báo khi **số lượng revision tăng nhanh**.

##### Nên tránh (Avoid) {#avoid}

* Sửa thủ công các đối tượng ControllerRevision.
* Dùng revision như một cơ chế sao lưu (hãy dùng các công cụ backup thực thụ).
* Đặt `revisionHistoryLimit: 0` (vô hiệu hóa khả năng rollback).

## Lưu giữ PersistentVolumeClaim (PersistentVolumeClaim retention) {#persistentvolumeclaim-retention}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

Trường tùy chọn `.spec.persistentVolumeClaimRetentionPolicy` kiểm soát việc các PVC
có bị xóa hay không và bị xóa như thế nào trong vòng đời của một StatefulSet. Bạn phải bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`StatefulSetAutoDeletePVC` trên API server và controller manager để dùng trường này.
Sau khi bật, có hai chính sách bạn có thể cấu hình cho mỗi StatefulSet:

`whenDeleted`
: Cấu hình hành vi lưu giữ volume áp dụng khi StatefulSet bị xóa.

`whenScaled`
: Cấu hình hành vi lưu giữ volume áp dụng khi số replica của
  StatefulSet bị giảm xuống; ví dụ, khi thu nhỏ (scale down) set.

Với mỗi chính sách mà bạn có thể cấu hình, bạn có thể đặt giá trị là `Delete` hoặc `Retain`.

`Delete`
: Các PVC được tạo từ `volumeClaimTemplate` của StatefulSet bị xóa cho mỗi Pod
  chịu ảnh hưởng của chính sách. Với chính sách `whenDeleted`, tất cả PVC từ
  `volumeClaimTemplate` bị xóa sau khi các Pod của chúng đã bị xóa. Với chính sách
  `whenScaled`, chỉ các PVC tương ứng với các replica Pod đang bị thu nhỏ mới bị
  xóa, sau khi các Pod của chúng đã bị xóa.

`Retain` (mặc định)
: Các PVC từ `volumeClaimTemplate` không bị ảnh hưởng khi Pod của chúng bị
  xóa. Đây là hành vi trước khi có tính năng mới này.

Hãy nhớ rằng các chính sách này **chỉ** áp dụng khi các Pod đang bị gỡ bỏ do
StatefulSet bị xóa hoặc bị thu nhỏ. Ví dụ, nếu một Pod gắn với một StatefulSet
bị lỗi do node hỏng, và control plane tạo một Pod thay thế, StatefulSet
giữ lại PVC hiện có. Volume hiện có không bị ảnh hưởng, và cluster sẽ gắn (attach) nó vào
node nơi Pod mới sắp khởi chạy.

Mặc định của các chính sách là `Retain`, khớp với hành vi của StatefulSet trước khi có tính năng mới này.

Đây là một ví dụ về chính sách:

```yaml
apiVersion: apps/v1
kind: StatefulSet
...
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
...
```

StatefulSet controller thêm
[owner reference](30-owners-dependents-vi.md#owner-references-in-object-specifications)
vào các PVC của nó, các PVC này sau đó bị bộ thu gom rác (garbage collector) xóa sau khi Pod kết thúc. Điều này cho phép Pod
unmount sạch sẽ tất cả các volume trước khi các PVC bị xóa (và trước khi PV và
volume phía sau bị xóa, tùy theo retain policy). Khi bạn đặt chính sách `whenDeleted`
là `Delete`, một owner reference tới instance StatefulSet được đặt trên tất cả các PVC
gắn với StatefulSet đó.

Chính sách `whenScaled` chỉ được xóa PVC khi một Pod bị thu nhỏ (scale down), chứ không phải khi một
Pod bị xóa vì lý do khác. Khi thực hiện đối chiếu (reconcile), StatefulSet controller so sánh
số replica mong muốn của nó với các Pod thực tế hiện diện trên cluster. Bất kỳ Pod StatefulSet nào
có id lớn hơn số replica đều bị kết án (condemned) và bị đánh dấu để xóa. Nếu chính sách
`whenScaled` là `Delete`, các Pod bị kết án trước tiên được đặt làm owner của các
PVC template liên quan của StatefulSet, trước khi Pod bị xóa. Điều này khiến các PVC
chỉ bị thu gom rác sau khi các Pod bị kết án đã kết thúc.

Điều này có nghĩa là nếu controller bị crash và khởi động lại, không Pod nào sẽ bị xóa trước khi
owner reference của nó được cập nhật phù hợp với chính sách. Nếu một Pod bị kết án bị
force-delete trong khi controller đang không hoạt động, owner reference có thể đã hoặc chưa được
thiết lập, tùy vào thời điểm controller bị crash. Có thể mất vài vòng lặp reconcile để
cập nhật các owner reference, vì vậy một số Pod bị kết án có thể đã được thiết lập owner reference còn
số khác thì chưa. Vì lý do này, chúng tôi khuyến nghị chờ controller hoạt động trở lại,
controller sẽ xác minh các owner reference trước khi kết thúc các Pod. Nếu điều đó là không thể,
người vận hành nên xác minh các owner reference trên các PVC để đảm bảo các đối tượng mong đợi bị
xóa khi các Pod bị force-delete.

### Replicas {#replicas}

`.spec.replicas` là một trường tùy chọn chỉ định số Pod mong muốn. Mặc định là 1.

Nếu bạn scale thủ công một StatefulSet, thông qua `kubectl scale
statefulset statefulset --replicas=X`, và sau đó bạn cập nhật StatefulSet đó
dựa trên một manifest (ví dụ: bằng cách chạy `kubectl apply -f
statefulset.yaml`), thì việc apply manifest đó sẽ ghi đè việc scale thủ công
mà bạn đã làm trước đó.

Nếu một [HorizontalPodAutoscaler](72-horizontal-pod-autoscale-vi.md)
(hoặc bất kỳ API tương tự nào cho việc mở rộng theo chiều ngang) đang quản lý việc scale cho một
StatefulSet, đừng đặt `.spec.replicas`. Thay vào đó, hãy để
control plane của Kubernetes quản lý trường `.spec.replicas` một cách tự động.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](46-pods-vi.md).
* Tìm hiểu cách sử dụng StatefulSet
  * Làm theo ví dụ về [triển khai một ứng dụng stateful](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/).
  * Làm theo ví dụ về [triển khai Cassandra với StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/).
  * Làm theo ví dụ về [chạy một ứng dụng stateful có nhân bản (replicated)](343-run-replicated-stateful-application-vi.md).
  * Tìm hiểu cách [scale một StatefulSet](347-scale-stateful-set-vi.md).
  * Tìm hiểu những gì liên quan khi bạn [xóa một StatefulSet](340-delete-stateful-set-vi.md).
  * Tìm hiểu cách [cấu hình một Pod dùng volume để lưu trữ](280-configure-volume-storage-vi.md).
  * Tìm hiểu cách [cấu hình một Pod dùng PersistentVolume để lưu trữ](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/).
* `StatefulSet` là một tài nguyên cấp cao nhất (top-level) trong Kubernetes REST API.
  Đọc định nghĩa đối tượng
  [StatefulSet](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/)
  để hiểu API cho stateful set.
* Đọc về [PodDisruptionBudget](53-disruptions-vi.md) và cách
  bạn có thể dùng nó để quản lý tính khả dụng của ứng dụng trong các gián đoạn (disruption).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. StatefulSet `web` 3 replica đang chạy trên hai worker của bạn. Bạn xóa `web-1` đang nằm
   trên `lab-k8s-worker2`. Pod thay thế mang tên gì, và nó có thể được lập lịch sang
   `lab-k8s-worker1` không?
2. `podManagementPolicy` để mặc định. `web-0` crash **sau khi** `web-1` đã Running và Ready
   nhưng **trước khi** `web-2` được khởi chạy. `web-2` khi nào mới lên?
3. **Câu bẫy.** Bạn thu nhỏ StatefulSet từ 3 xuống 1. Pod nào bị kết thúc trước, và các
   volume gắn với những Pod đó có bị xóa theo không?
4. Bạn đổi image sang một tag không tồn tại. Rollout dừng ở Pod có ordinal nào, và vì sao chỉ
   sửa lại Pod template là chưa đủ để cứu?
5. Ứng dụng của bạn không cần định danh ổn định, cũng không cần thứ tự khi triển khai hay
   thu nhỏ. Bài khuyên dùng gì, và vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod thay thế vẫn tên là **`web-1`**, và **có**, nó hoàn toàn có thể chạy trên
   `lab-k8s-worker1`. Đây là điểm cốt lõi của mục *Định danh của Pod*: định danh gồm số thứ tự,
   định danh mạng ổn định và lưu trữ ổn định, và nó "gắn chặt với Pod, **bất kể Pod được lập
   lịch (lại) lên node nào**". Định danh ổn định **không** có nghĩa là Pod bị ghim vào một
   node — nó có nghĩa là tên, hostname và các claim đi theo ordinal chứ không đi theo node.
2. **`web-2` không được khởi chạy cho tới khi `web-0` được khởi động lại thành công và trở
   thành Running và Ready.** Bài nêu đúng tình huống này. Quy tắc chung của `OrderedReady`:
   trước khi một thao tác mở rộng được áp dụng cho một Pod, **tất cả** các Pod đứng trước nó
   phải Running và Ready — không chỉ Pod liền kề. Nếu `.spec.minReadySeconds` được đặt thì
   các Pod đứng trước còn phải Ready đủ số giây đó.
3. **`web-2` bị kết thúc trước**, và `web-1` chỉ bị kết thúc sau khi `web-2` đã tắt hoàn toàn
   và bị xóa — Pod bị kết thúc theo thứ tự ngược, từ N-1 về 0, và trước khi một Pod bị kết
   thúc thì mọi Pod đứng sau nó phải đã shutdown hoàn toàn. Còn volume thì **không bị xóa**:
   mục *Hạn chế* nói thẳng "việc xóa và/hoặc thu nhỏ một StatefulSet sẽ _không_ xóa các
   volume gắn với StatefulSet đó", vì an toàn dữ liệu được coi trọng hơn việc tự động dọn
   sạch tài nguyên. Trực giác "scale down thì dọn sạch theo" là sai ở đây, và đây đúng là chỗ
   khác Deployment nhiều nhất.
4. Rollout dừng ở **ordinal lớn nhất**, tức Pod đầu tiên được cập nhật (`RollingUpdate` của
   StatefulSet đi từ số thứ tự lớn nhất xuống nhỏ nhất) — nó không bao giờ Running và Ready
   nên control plane chờ mãi và không đụng tới các Pod đứng trước. Sửa template là chưa đủ vì
   một **vấn đề đã biết**: StatefulSet vẫn tiếp tục chờ Pod hỏng trở thành Ready (điều không
   bao giờ xảy ra) trước khi thử hoàn nguyên Pod đó. Bạn phải **xóa tay mọi Pod đã chạy với
   cấu hình lỗi**, sau đó StatefulSet mới bắt đầu tạo lại bằng template đã hoàn nguyên.
5. Bài khuyên dùng **một đối tượng workload cung cấp tập replica stateless — Deployment hoặc
   ReplicaSet**. Lý do: các bảo đảm của StatefulSet (định danh cố định, thứ tự tuần tự) không
   miễn phí — chúng làm việc triển khai và thu nhỏ chậm hơn hẳn, thêm ràng buộc phải tự tạo
   Headless Service, và mở ra trạng thái hỏng cần can thiệp thủ công ở mục *Rollback bắt
   buộc*. Không cần thì đừng trả giá đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
