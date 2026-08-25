# Pod (Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 2/11 · Kiểm chứng
ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md).

Bài này là **trang mục lục của cả nhóm 3a**: nó nêu tên init container, sidecar, ephemeral
container, probe, static Pod, requests/limits, `securityContext` rồi trỏ sang chỗ khác. Đừng cố
hiểu sâu những đoạn đó ở đây — mỗi thứ có một bài riêng ngay sau. Phần cốt lõi chỉ là mô hình
Pod: chia sẻ cái gì, sống bao lâu, và sửa được gì.

**Phải hiểu ở lần đọc này:**

- Pod là **đơn vị triển khai nhỏ nhất**; ngữ cảnh dùng chung của nó là một tập Linux namespace và
  cgroup, nên nội dung một Pod luôn được đặt cùng vị trí và lập lịch cùng nhau.
- Mạng của Pod: mỗi Pod một địa chỉ IP, mọi container trong Pod chung network namespace nên gọi
  nhau bằng `localhost` và **phải tự chia nhau không gian port**; container ở Pod khác thì chỉ
  đến được qua mạng IP.
- Pod là thực thể **phù du và không phải một tiến trình**: nó ở lại đúng node đã được lập lịch
  cho tới khi chạy xong, bị xóa, bị trục xuất, hoặc node hỏng. Khởi động lại một container
  **không phải** là khởi động lại Pod.
- Pod gần như bất biến. Hầu hết metadata không sửa được, và cập nhật thông thường chỉ đổi được
  `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
  `spec.terminationGracePeriodSeconds`, `spec.tolerations` và `spec.schedulingGates`. Muốn đổi
  thứ khác thì phải thay Pod — đó chính là lý do tồn tại của pod template và controller.
- Nhân bản nghĩa là **thêm Pod**, không phải thêm container: mỗi Pod chạy một thực thể ứng dụng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Init container, sidecar, ephemeral container, probe — ở đây chỉ nêu tên | mỗi thứ có bài riêng trong chính nhóm này | bài [49](49-probes-vi.md), [50](50-init-containers-vi.md), [51](51-sidecar-containers-vi.md), [52](52-ephemeral-containers-vi.md) |
| *Yêu cầu và giới hạn tài nguyên* | ở đây chỉ tóm tắt vài dòng | giai đoạn 3, nhóm 3b — bài [110](110-manage-resources-containers-vi.md) |
| *Thiết lập bảo mật cho Pod* (`securityContext`) | mới chỉ nêu tên trường | bài [60](60-advanced-pod-config-vi.md) cuối nhóm này, rồi giai đoạn 9 |
| *Static Pod* | gắn với cách kubeadm chạy control plane | giai đoạn 3, nhóm 3b — bài [58](58-static-pods-vi.md) |
| *Subresource của Pod* và *Generation của Pod* (`observedGeneration`) | chi tiết API, chưa cần để tạo Pod | phần `resize` ở bài [47](47-pod-lifecycle-vi.md); `observedGeneration` ở bài [48](48-pod-condition-vi.md) |
| *Hệ điều hành của Pod* (`.spec.os.name`) | chỉ có nghĩa khi cluster có node Windows | giai đoạn 15 |
| *Chỉ định nhóm lập lịch* (`spec.schedulingGroup`) | tính năng alpha, cần PodGroup | giai đoạn 13 — bài [59](59-scheduling-group-vi.md) |

---

_Pod_ là đơn vị tính toán nhỏ nhất có thể triển khai mà bạn có thể tạo và quản lý trong Kubernetes.

Một _Pod_ (như một đàn cá voi — pod of whales — hay một quả đậu — pea pod) là một nhóm gồm một hoặc nhiều
container, với tài nguyên lưu trữ và mạng dùng chung, cùng một đặc tả (specification) về cách chạy
các container đó. Nội dung của một Pod luôn được đặt cùng vị trí (co-located) và
được lập lịch cùng nhau (co-scheduled), và chạy trong một ngữ cảnh dùng chung. Một Pod mô hình hóa
một "máy chủ logic" (logical host) đặc thù cho ứng dụng: nó chứa một hoặc nhiều container ứng dụng
gắn kết tương đối chặt chẽ với nhau.
Trong bối cảnh ngoài đám mây (non-cloud), các ứng dụng chạy trên cùng một máy vật lý hoặc máy ảo
tương tự như các ứng dụng đám mây chạy trên cùng một máy chủ logic.

Bên cạnh các container ứng dụng, một Pod có thể chứa
các init container chạy trong quá trình khởi động Pod. Bạn cũng có thể tiêm (inject)
các ephemeral container để gỡ lỗi (debug) một Pod đang chạy.

## Pod là gì? (What is a Pod?)

> **Ghi chú:**
> Bạn cần cài đặt một [container runtime](./00-container-runtimes-vi.md)
> vào từng node trong cluster để các Pod có thể chạy ở đó.

Ngữ cảnh dùng chung của một Pod là một tập các Linux namespace, cgroup, và
tiềm năng là những khía cạnh cách ly khác — chính là những thứ cách ly một container.
Bên trong ngữ cảnh của một Pod, các ứng dụng riêng lẻ có thể được áp dụng
thêm các mức cách ly con.

Một Pod tương tự như một tập các container có chung namespace và các filesystem volume dùng chung.

Pod trong một cluster Kubernetes được dùng theo hai cách chính:

* **Pod chạy một container duy nhất**. Mô hình "một container mỗi Pod" là
  trường hợp sử dụng phổ biến nhất của Kubernetes; trong trường hợp này, bạn có thể coi Pod như
  một lớp bọc (wrapper) quanh một container duy nhất; Kubernetes quản lý các Pod thay vì
  quản lý trực tiếp các container.
* **Pod chạy nhiều container cần phối hợp với nhau**. Một Pod có thể
  đóng gói một ứng dụng gồm
  [nhiều container đặt cùng vị trí](#how-pods-manage-multiple-containers)
  gắn kết chặt chẽ và cần chia sẻ tài nguyên. Các container đặt cùng vị trí này
  tạo thành một đơn vị gắn kết duy nhất.

  Việc nhóm nhiều container đặt cùng vị trí và được quản lý cùng nhau trong một Pod duy nhất là
  một trường hợp sử dụng tương đối nâng cao. Bạn chỉ nên dùng mẫu (pattern) này trong những
  tình huống cụ thể mà các container của bạn gắn kết chặt chẽ với nhau.

  Bạn không cần chạy nhiều container để cung cấp khả năng nhân bản (nhằm tăng khả năng chống chịu
  hoặc dung lượng); nếu bạn cần nhiều bản sao (replica), hãy xem
  [Quản lý workload](62-controllers-index-vi.md).

## Sử dụng Pod (Using Pods)

Dưới đây là ví dụ về một Pod gồm một container chạy image `nginx:1.14.2`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

Để tạo Pod ở trên, chạy lệnh sau:
```shell
kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml
```

Pod thường không được tạo trực tiếp mà được tạo thông qua các tài nguyên workload.
Xem [Làm việc với Pod](#working-with-pods) để biết thêm thông tin về cách Pod được dùng
cùng các tài nguyên workload.

### Tài nguyên workload để quản lý Pod (Workload resources for managing pods)

Thông thường bạn không cần tạo Pod trực tiếp, kể cả các Pod đơn lẻ (singleton).
Thay vào đó, hãy tạo chúng bằng các tài nguyên workload như Deployment hoặc Job.
Nếu các Pod của bạn cần theo dõi trạng thái, hãy cân nhắc tài nguyên StatefulSet.

Mỗi Pod được thiết kế để chạy một thực thể (instance) duy nhất của một ứng dụng cho trước. Nếu bạn muốn
mở rộng ứng dụng theo chiều ngang (cung cấp thêm tài nguyên tổng thể bằng cách chạy
nhiều thực thể hơn), bạn nên dùng nhiều Pod, mỗi Pod cho một thực thể. Trong
Kubernetes, điều này thường được gọi là _nhân bản_ (replication).
Các Pod được nhân bản thường được tạo và quản lý theo nhóm bởi một tài nguyên workload
và controller của nó.

Xem [Pod và controller](#pods-and-controllers) để biết thêm thông tin về cách
Kubernetes dùng các tài nguyên workload, cùng các controller của chúng, để hiện thực việc
mở rộng (scaling) và tự phục hồi (auto-healing) ứng dụng.

Pod cung cấp sẵn hai loại tài nguyên dùng chung cho các container thành phần của chúng:
[mạng](#pod-networking) và [lưu trữ](#pod-storage).

## Làm việc với Pod (Working with Pods) {#working-with-pods}

Bạn sẽ hiếm khi tạo trực tiếp từng Pod riêng lẻ trong Kubernetes — kể cả các Pod đơn lẻ. Lý do là
Pod được thiết kế như những thực thể tương đối phù du (ephemeral), dùng xong rồi bỏ. Khi
một Pod được tạo ra (trực tiếp bởi bạn, hoặc gián tiếp bởi một
controller), Pod mới được lập lịch chạy trên một node trong cluster của bạn.
Pod ở lại trên node đó cho đến khi Pod chạy xong, đối tượng Pod bị xóa,
Pod bị *trục xuất* (evicted) vì thiếu tài nguyên, hoặc node gặp sự cố.

> **Ghi chú:**
> Không nên nhầm lẫn việc khởi động lại một container trong Pod với việc khởi động lại một Pod. Pod
> không phải là một tiến trình (process), mà là một môi trường để chạy (các) container. Một Pod tồn tại
> cho đến khi nó bị xóa.

Tên của một Pod phải là một giá trị
[DNS subdomain](17-names-vi.md#dns-subdomain-names)
hợp lệ, nhưng điều này có thể tạo ra kết quả không mong đợi cho hostname của Pod. Để có tính tương thích
tốt nhất, tên nên tuân theo quy tắc chặt chẽ hơn dành cho
[DNS label](17-names-vi.md#dns-label-names).

### Hệ điều hành của Pod (Pod OS)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Bạn nên đặt trường `.spec.os.name` thành `windows` hoặc `linux` để chỉ ra hệ điều hành
mà các container trong Pod đó yêu cầu. Đây là hai hệ điều hành duy nhất hiện được
Kubernetes hỗ trợ. Trong tương lai, danh sách này có thể được mở rộng.

Kubelet từ chối chạy một Pod nếu giá trị của `.spec.os.name` không khớp với
hệ điều hành của node. Tuy nhiên, trong Kubernetes
v1.36, giá trị của `.spec.os.name` không ảnh hưởng đến
cách kube-scheduler
chọn node để chạy Pod. Trong bất kỳ cluster nào có nhiều hơn một hệ điều hành cho
các node đang chạy, bạn nên đặt nhãn
[kubernetes.io/os](https://kubernetes.io/docs/reference/labels-annotations-taints/#kubernetes-io-os)
chính xác trên từng node, và định nghĩa các Pod với `nodeSelector` dựa trên nhãn hệ điều hành.
Kube-scheduler gán Pod của bạn cho một node dựa trên các tiêu chí khác và có thể thành công hoặc không
trong việc chọn được vị trí node phù hợp có hệ điều hành đúng với các container trong Pod đó.
[Các tiêu chuẩn bảo mật Pod (Pod security standards)](115-pod-security-standards-vi.md) cũng dùng trường
này để tránh áp đặt các chính sách không liên quan đến hệ điều hành đó.

### Pod và controller (Pods and controllers) {#pods-and-controllers}

Bạn có thể dùng các tài nguyên workload để tạo và quản lý nhiều Pod thay cho bạn. Một controller
của tài nguyên đó xử lý việc nhân bản, triển khai bản cập nhật (rollout), và tự động phục hồi khi
Pod gặp sự cố. Ví dụ, nếu một Node gặp sự cố, controller nhận thấy các Pod trên
Node đó đã ngừng hoạt động và tạo Pod thay thế. Scheduler đặt
Pod thay thế lên một Node khỏe mạnh.

Dưới đây là một số ví dụ về tài nguyên workload quản lý một hoặc nhiều Pod:

* Deployment
* StatefulSet
* DaemonSet

### Chỉ định nhóm lập lịch (Specifying a scheduling group)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Theo mặc định, Kubernetes lập lịch từng Pod một cách riêng lẻ. Tuy nhiên, một số ứng dụng gắn kết chặt chẽ
cần một nhóm Pod được lập lịch đồng thời thì mới hoạt động đúng.

Bạn có thể liên kết một Pod với một [PodGroup](75-podgroup-api-vi.md) bằng trường
[scheduling group](59-scheduling-group-vi.md)
(`spec.schedulingGroup`). Trường này cho `kube-scheduler` biết rằng `Pod` thuộc về một nhóm
cụ thể, cho phép nó áp dụng các quyết định sắp đặt phối hợp ở cấp nhóm cho toàn bộ nhóm cùng một lúc.

### Pod template

Các controller của tài nguyên workload tạo Pod
từ một _pod template_ và quản lý các Pod đó thay cho bạn.

PodTemplate là các đặc tả để tạo Pod, và được bao gồm trong các tài nguyên workload như
[Deployment](63-deployment-vi.md),
[Job](67-job-vi.md), và
[DaemonSet](66-daemonset-vi.md).

Mỗi controller của một tài nguyên workload dùng `PodTemplate` bên trong đối tượng workload
để tạo ra các Pod thực tế. `PodTemplate` là một phần của trạng thái mong muốn (desired state) của bất kỳ
tài nguyên workload nào mà bạn đã dùng để chạy ứng dụng của mình.

Khi tạo một Pod, bạn có thể bao gồm các
[biến môi trường](331-define-environment-variable-vi.md)
trong pod template cho các container chạy trong Pod.

Mẫu dưới đây là manifest cho một Job đơn giản với một `template` khởi động một
container. Container trong Pod đó in ra một thông điệp rồi tạm dừng.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # Đây là pod template
    spec:
      containers:
      - name: hello
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # Pod template kết thúc ở đây
```

Việc sửa đổi pod template hoặc chuyển sang một pod template mới không có tác động trực tiếp
đến các Pod đã tồn tại. Nếu bạn thay đổi pod template của một tài nguyên
workload, tài nguyên đó cần tạo các Pod thay thế sử dụng template đã cập nhật.

Ví dụ, StatefulSet controller đảm bảo rằng các Pod đang chạy khớp với pod template
hiện tại của từng đối tượng StatefulSet. Nếu bạn chỉnh sửa StatefulSet để thay đổi pod
template của nó, StatefulSet bắt đầu tạo các Pod mới dựa trên template đã cập nhật.
Cuối cùng, tất cả các Pod cũ được thay thế bằng các Pod mới, và việc cập nhật hoàn tất.

Mỗi tài nguyên workload hiện thực các quy tắc riêng để xử lý những thay đổi đối với pod template.
Nếu bạn muốn đọc thêm cụ thể về StatefulSet, hãy đọc
[Chiến lược cập nhật (Update strategy)](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets) trong bài hướng dẫn StatefulSet Basics.

Trên các Node, kubelet không
trực tiếp quan sát hay quản lý bất kỳ chi tiết nào xoay quanh pod template và việc cập nhật chúng; những
chi tiết đó đã được trừu tượng hóa. Sự trừu tượng hóa và tách bạch mối quan tâm (separation of concerns) đó
giúp đơn giản hóa ngữ nghĩa của hệ thống, và giúp việc mở rộng hành vi của cluster trở nên khả thi
mà không cần thay đổi mã nguồn hiện có.

## Cập nhật và thay thế Pod (Pod update and replacement)

Như đã đề cập ở phần trước, khi pod template của một tài nguyên workload
bị thay đổi, controller tạo các Pod mới dựa trên template đã cập nhật
thay vì cập nhật hoặc vá (patch) các Pod hiện có.

Kubernetes không ngăn bạn quản lý Pod trực tiếp. Bạn có thể
cập nhật tại chỗ một số trường của một Pod đang chạy. Tuy nhiên, các thao tác cập nhật Pod
như
[`patch`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#patch-pod-v1-core) và
[`replace`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#replace-pod-v1-core)
có một số hạn chế:

- Hầu hết metadata của một Pod là bất biến (immutable). Ví dụ, bạn không thể
  thay đổi các trường `namespace`, `name`, `uid`, hay `creationTimestamp`.

- Nếu `metadata.deletionTimestamp` đã được đặt, không thể thêm mục mới vào
  danh sách `metadata.finalizers`.
- Các cập nhật Pod không được thay đổi các trường khác ngoài `spec.containers[*].image`,
  `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`, `spec.terminationGracePeriodSeconds`,
  `spec.tolerations` hoặc `spec.schedulingGates`. Với `spec.tolerations`, bạn chỉ có thể thêm mục mới.
- Khi cập nhật trường `spec.activeDeadlineSeconds`, hai loại cập nhật
  được cho phép:

  1. đặt trường chưa được gán giá trị thành một số dương;
  2. cập nhật trường từ một số dương thành một số nhỏ hơn, không âm.

### Subresource của Pod (Pod subresources)

Các quy tắc cập nhật ở trên áp dụng cho các cập nhật Pod thông thường, nhưng những trường khác của Pod có thể được cập nhật thông qua các _subresource_.

- **Resize:** Subresource `resize` cho phép cập nhật tài nguyên của container (`spec.containers[*].resources`).
  Xem [Resize Container Resources](289-resize-container-resources-vi.md) để biết thêm chi tiết.
- **Ephemeral Containers:** Subresource `ephemeralContainers` cho phép
  thêm các ephemeral container
  vào một Pod.
  Xem [Ephemeral Containers](52-ephemeral-containers-vi.md) để biết thêm chi tiết.
- **Status:** Subresource `status` cho phép cập nhật trạng thái của Pod.
  Subresource này thường chỉ được dùng bởi kubelet và các controller hệ thống khác.
- **Binding:** Subresource `binding` cho phép đặt `spec.nodeName` của Pod thông qua một yêu cầu `Binding`.
  Subresource này thường chỉ được dùng bởi scheduler.

### Generation của Pod (Pod generation) {#pod-generation}

- Trường `metadata.generation` là duy nhất. Nó sẽ được hệ thống tự động đặt
  sao cho các Pod mới có `metadata.generation` bằng 1, và mỗi lần cập nhật
  các trường khả biến (mutable) trong spec của Pod sẽ tăng `metadata.generation` thêm 1.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

- `observedGeneration` là một trường được ghi nhận trong phần `status` của đối tượng Pod.
  Kubelet sẽ đặt `status.observedGeneration`
  để gắn trạng thái của Pod với pod status hiện tại. `status.observedGeneration` của Pod sẽ phản ánh
  `metadata.generation` của Pod tại thời điểm pod status được báo cáo.

> **Ghi chú:**
> Trường `status.observedGeneration` được quản lý bởi kubelet và các controller bên ngoài **không** được sửa đổi trường này.

Các trường status khác nhau có thể được gắn với `metadata.generation` của vòng lặp đồng bộ (sync loop) hiện tại, hoặc với
`metadata.generation` của vòng lặp đồng bộ trước đó. Điểm phân biệt chính là liệu một thay đổi trong `spec` được phản ánh
trực tiếp vào `status` hay là kết quả gián tiếp của một tiến trình đang chạy.

#### Cập nhật status trực tiếp (Direct Status Updates)

Với các trường status mà phần spec đã cấp phát được phản ánh trực tiếp, `observedGeneration` sẽ
được gắn với `metadata.generation` hiện tại (Generation N).

Hành vi này áp dụng cho:

- **Resize Status**: Trạng thái của một thao tác thay đổi kích thước (resize) tài nguyên.
- **Allocated Resources**: Các tài nguyên được cấp phát cho Pod sau một lần resize.
- **Ephemeral Containers**: Khi một ephemeral container mới được thêm vào, và nó đang ở trạng thái `Waiting`.

#### Cập nhật status gián tiếp (Indirect Status Updates)

Với các trường status là kết quả gián tiếp của việc thực thi spec, `observedGeneration` sẽ được gắn
với `metadata.generation` của vòng lặp đồng bộ trước đó (Generation N-1).

Hành vi này áp dụng cho:

- **Container Image**: `ContainerStatus.ImageID` phản ánh image của generation trước cho đến khi image mới
  được kéo về (pull) và container được cập nhật.
- **Actual Resources**: Trong khi một thao tác resize đang diễn ra, tài nguyên thực tế đang sử dụng vẫn thuộc về
  yêu cầu của generation trước.
- **Container state**: Trong khi một thao tác resize đang diễn ra, với chính sách yêu cầu khởi động lại (require restart policy),
  trạng thái container phản ánh yêu cầu của generation trước.
- **activeDeadlineSeconds** & **terminationGracePeriodSeconds** & **deletionTimestamp**: Ảnh hưởng của các trường này lên
  status của Pod là kết quả của đặc tả đã được quan sát trước đó.

## Chia sẻ tài nguyên và giao tiếp (Resource sharing and communication)

Pod cho phép chia sẻ dữ liệu và giao tiếp giữa các container
thành phần của chúng.

### Lưu trữ trong Pod (Storage in Pods) {#pod-storage}

Một Pod có thể chỉ định một tập các volume
lưu trữ dùng chung. Tất cả các container
trong Pod đều có thể truy cập các volume dùng chung này, cho phép các container đó
chia sẻ dữ liệu. Volume cũng cho phép dữ liệu lâu dài trong một Pod tồn tại
qua trường hợp một trong các container bên trong cần được khởi động lại. Xem
[Storage](90-storage-vi.md) để biết thêm thông tin về cách
Kubernetes hiện thực lưu trữ dùng chung và cung cấp nó cho các Pod.

### Mạng của Pod (Pod networking) {#pod-networking}

Mỗi Pod được gán một địa chỉ IP duy nhất cho mỗi họ địa chỉ (address family). Mọi
container trong một Pod chia sẻ chung network namespace, bao gồm địa chỉ IP và
các port mạng. Bên trong một Pod (và **chỉ** khi đó), các container thuộc về Pod
có thể giao tiếp với nhau bằng `localhost`. Khi các container trong một Pod giao tiếp
với các thực thể *bên ngoài Pod*,
chúng phải phối hợp cách sử dụng các tài nguyên mạng dùng chung (chẳng hạn như port).
Trong một Pod, các container chia sẻ một địa chỉ IP và không gian port, và
có thể tìm thấy nhau qua `localhost`. Các container trong một Pod cũng có thể giao tiếp
với nhau bằng các cơ chế giao tiếp liên tiến trình (inter-process communication) tiêu chuẩn như
SystemV semaphore hay POSIX shared memory. Các container ở những Pod khác nhau có địa chỉ IP
khác nhau và không thể giao tiếp bằng IPC mức hệ điều hành nếu không có cấu hình đặc biệt.
Các container muốn tương tác với một container chạy trong Pod khác có thể
dùng mạng IP để giao tiếp.

Các container bên trong Pod thấy hostname của hệ thống giống với
`name` được cấu hình cho Pod. Có thêm thông tin về điều này trong phần
[mạng](157-networking-vi.md).

## Thiết lập bảo mật cho Pod (Pod security settings) {#pod-security}

Để đặt các ràng buộc bảo mật lên Pod và container, bạn dùng trường
`securityContext` trong đặc tả của Pod. Trường này cho bạn
quyền kiểm soát chi tiết những gì một Pod hoặc từng container riêng lẻ có thể làm. Xem [Advanced Pod Configuration](60-advanced-pod-config-vi.md) để biết thêm chi tiết.

Để cấu hình bảo mật cơ bản, bạn nên đáp ứng tiêu chuẩn bảo mật Pod mức Baseline và chạy container không dùng quyền root (non-root). Bạn có thể đặt các security context đơn giản:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: ["sh", "-c", "sleep 1h"]
```

Để cấu hình security context nâng cao bao gồm capabilities, seccomp profile, và các tùy chọn bảo mật chi tiết, xem phần [các khái niệm bảo mật](113-security-vi.md).

* Để tìm hiểu về các ràng buộc bảo mật mức kernel mà bạn có thể sử dụng,
  xem [Linux kernel security constraints for Pods and containers](127-linux-kernel-security-vi.md).
* Để tìm hiểu thêm về security context của Pod, xem
  [Configure a Security Context for a Pod or Container](291-security-context-vi.md).

## Yêu cầu và giới hạn tài nguyên (Resource requests and limits)

Khi chỉ định một Pod, bạn có thể tùy chọn chỉ định lượng tài nguyên mỗi loại
mà một container cần. Các tài nguyên phổ biến nhất để chỉ định là CPU và bộ nhớ (RAM).

Khi bạn chỉ định _yêu cầu_ tài nguyên (resource request) cho các container trong một Pod,
kube-scheduler dùng thông tin này để quyết định đặt Pod lên node nào.
Khi bạn chỉ định _giới hạn_ tài nguyên (resource limit) cho một container, kubelet thực thi
các giới hạn đó để container đang chạy không được phép dùng nhiều hơn
giới hạn tài nguyên mà bạn đã đặt.

Giới hạn CPU được thực thi bằng cơ chế điều tiết CPU (CPU throttling). Khi một container tiến gần đến
giới hạn CPU của nó, kernel hạn chế quyền truy cập CPU của container đó. Giới hạn bộ nhớ được thực thi
bởi kernel bằng cách kill do hết bộ nhớ (out-of-memory — OOM) khi một container vượt quá giới hạn.

> **Ghi chú:**
> Việc đặt giới hạn CPU đi kèm một sự đánh đổi. Giới hạn CPU giúp ngăn vấn đề "hàng xóm ồn ào"
> (noisy neighbor), khi một workload đơn lẻ làm cạn kiệt tài nguyên của các workload khác trên cùng node. Điều này
> đặc biệt quan trọng trong môi trường đa người thuê (multi-tenant). Tuy nhiên, giới hạn CPU có thể gây
> điều tiết ngay cả khi node còn dư năng lực CPU, có khả năng làm giảm
> hiệu năng của các workload nhạy cảm với độ trễ. Việc có đặt giới hạn CPU hay không phụ thuộc vào
> môi trường của bạn, đặc điểm của workload, và các yêu cầu cách ly.

Để biết chi tiết về đơn vị tài nguyên, hành vi thực thi, và các ví dụ cấu hình,
xem [Resource Management for Pods and Containers](110-manage-resources-containers-vi.md).

## Static Pod (Static Pods) {#static-pods}

_Static Pod_ được quản lý trực tiếp bởi kubelet daemon trên một node cụ thể,
mà không có sự quan sát của API server.
Trong khi hầu hết các Pod được quản lý bởi control plane (ví dụ, một
Deployment), với các static
Pod, kubelet trực tiếp giám sát từng static Pod (và khởi động lại nó nếu nó gặp sự cố).

Static Pod luôn gắn với một kubelet trên một node cụ thể.
Trường hợp sử dụng chính của static Pod là chạy một control plane tự lưu trữ (self-hosted): nói cách khác,
dùng kubelet để giám sát từng [thành phần của control plane](./22-architecture-vi.md#control-plane-components).

Để biết chi tiết, xem [Static Pods](58-static-pods-vi.md).

## Pod với nhiều container (Pods with multiple containers) {#how-pods-manage-multiple-containers}

Pod được thiết kế để hỗ trợ nhiều tiến trình hợp tác (dưới dạng container) tạo thành
một đơn vị dịch vụ gắn kết. Các container trong một Pod được tự động đặt cùng vị trí và
lập lịch cùng nhau trên cùng một máy vật lý hoặc máy ảo trong cluster. Các container
có thể chia sẻ tài nguyên và các phần phụ thuộc, giao tiếp với nhau, và phối hợp
thời điểm cũng như cách thức chúng bị chấm dứt.

Pod trong một cluster Kubernetes được dùng theo hai cách chính:

* **Pod chạy một container duy nhất**. Mô hình "một container mỗi Pod" là
  trường hợp sử dụng phổ biến nhất của Kubernetes; trong trường hợp này, bạn có thể coi Pod như
  một lớp bọc quanh một container duy nhất; Kubernetes quản lý các Pod thay vì
  quản lý trực tiếp các container.
* **Pod chạy nhiều container cần phối hợp với nhau**. Một Pod có thể
  đóng gói một ứng dụng gồm
  nhiều container đặt cùng vị trí,
  gắn kết chặt chẽ và cần chia sẻ tài nguyên. Các container đặt cùng vị trí này
  tạo thành một đơn vị dịch vụ gắn kết duy nhất — ví dụ, một container phục vụ ra bên ngoài
  dữ liệu lưu trong một volume dùng chung, trong khi một
  sidecar container riêng biệt
  làm mới hoặc cập nhật các file đó.
  Pod bọc các container này, các tài nguyên lưu trữ, và một danh tính mạng
  phù du (ephemeral network identity) lại với nhau thành một đơn vị duy nhất.

Ví dụ, bạn có thể có một container đóng vai trò
web server cho các file trong một volume dùng chung, và một
[sidecar container](51-sidecar-containers-vi.md) riêng biệt
cập nhật các file đó từ một nguồn ở xa, như trong sơ đồ sau:

![Sơ đồ tạo Pod](https://kubernetes.io/images/docs/pod.svg)

Một số Pod có các init container
cũng như các app container.
Theo mặc định, init container chạy và hoàn tất trước khi các app container được khởi động.

Bạn cũng có thể có các [sidecar container](51-sidecar-containers-vi.md)
cung cấp các dịch vụ phụ trợ cho Pod ứng dụng chính (ví dụ: một service mesh).

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Được bật theo mặc định, [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `SidecarContainers`
cho phép bạn chỉ định `restartPolicy: Always` cho các init container.
Việc đặt chính sách khởi động lại `Always` đảm bảo các container mà bạn đặt nó được
coi là _sidecar_ và được giữ chạy trong suốt toàn bộ vòng đời của Pod.
Các container mà bạn định nghĩa tường minh là sidecar container
sẽ khởi động trước Pod ứng dụng chính và tiếp tục chạy cho đến khi Pod
bị tắt.

## Probe của container (Container probes)

_Probe_ là một hoạt động chẩn đoán được kubelet thực hiện định kỳ trên một container.
Để thực hiện chẩn đoán, kubelet có thể gọi các hành động khác nhau:

- `ExecAction` (thực hiện với sự trợ giúp của container runtime)
- `TCPSocketAction` (được kubelet kiểm tra trực tiếp)
- `HTTPGetAction` (được kubelet kiểm tra trực tiếp)

Bạn có thể đọc thêm về [probe](47-pod-lifecycle-vi.md#container-probes)
trong tài liệu Pod Lifecycle.

## Tiếp theo (What's next)

* Tìm hiểu về [vòng đời của một Pod](47-pod-lifecycle-vi.md).
* Đọc về [PodDisruptionBudget](53-disruptions-vi.md)
  và cách bạn có thể dùng nó để quản lý tính khả dụng của ứng dụng trong các gián đoạn (disruption).
* Pod là một tài nguyên cấp cao nhất trong Kubernetes REST API.
  Định nghĩa đối tượng [Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
  mô tả chi tiết đối tượng này.
* [The Distributed System Toolkit: Patterns for Composite Containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/) giải thích các bố cục phổ biến cho các Pod có nhiều hơn một container.
* Đọc về [các ràng buộc phân bố theo topology của Pod (Pod topology spread constraints)](140-topology-spread-constraints-vi.md)
* Đọc [Advanced Pod Configuration](60-advanced-pod-config-vi.md) để tìm hiểu chủ đề này một cách chi tiết.
  Trang đó đề cập đến các khía cạnh cấu hình Pod vượt ra ngoài những điều cốt lõi, bao gồm:
  * PriorityClasses
  * RuntimeClasses
  * các cách nâng cao để cấu hình _lập lịch_ (scheduling): cách mà Kubernetes quyết định Pod nên chạy trên node nào.

Để hiểu bối cảnh vì sao Kubernetes bọc một Pod API chung trong các tài nguyên khác
(chẳng hạn StatefulSet hay
Deployment),
bạn có thể đọc về các công trình đi trước, bao gồm:

* [Aurora](https://aurora.apache.org/documentation/latest/reference/configuration/#job-schema)
* [Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/)
* [Marathon](https://github.com/d2iq-archive/marathon)
* [Omega](https://research.google/pubs/pub41684/)
* [Tupperware](https://engineering.fb.com/data-center-engineering/tupperware/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Hai container trong cùng một Pod gọi nhau bằng địa chỉ nào, và điều gì chúng buộc phải tự
   thỏa thuận với nhau?
2. Trên cluster lab bạn cần ba bản sao của một web server. Tạo một Pod có ba container nginx,
   hay ba Pod mỗi Pod một container? Bài lập luận thế nào?
3. Bạn `kubectl edit` một Pod đang chạy: đổi `image` của container thì được, còn thêm
   `nodeSelector` thì bị từ chối. Vì sao?
4. Một container trong Pod crash rồi được khởi động lại. Pod đó có được coi là đã khởi động lại
   không, và nó có thể chuyển sang node khác không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bằng **`localhost`**. Mọi container trong một Pod **chia sẻ chung network namespace**, tức là
   chung địa chỉ IP và chung không gian port — nên thứ chúng phải tự thỏa thuận là **port**: hai
   container trong cùng Pod không thể cùng nghe trên một port. Bên trong Pod (và **chỉ** khi đó)
   mới dùng được `localhost`; container ở Pod khác có IP khác và phải đi qua mạng IP.
2. **Ba Pod, mỗi Pod một container.** Mỗi Pod được thiết kế để chạy **một thực thể duy nhất** của
   một ứng dụng; muốn mở rộng theo chiều ngang thì dùng nhiều Pod, mỗi Pod một thực thể — đó là
   _nhân bản_. Bài nói thẳng: bạn **không cần** chạy nhiều container để có khả năng nhân bản.
   Nhiều container trong một Pod là mẫu dành cho các thành phần gắn kết chặt và chia sẻ tài
   nguyên, và là một trường hợp sử dụng tương đối nâng cao.
3. Vì cập nhật Pod **không được thay đổi các trường khác ngoài** `spec.containers[*].image`,
   `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
   `spec.terminationGracePeriodSeconds`, `spec.tolerations` và `spec.schedulingGates`.
   `image` nằm trong danh sách, `nodeSelector` thì không. **Pod gần như bất biến**; muốn đổi thứ
   ngoài danh sách đó thì phải thay Pod, và đó là việc của controller qua pod template.
4. **Không, và không.** Bài cảnh báo đừng nhầm khởi động lại một container với khởi động lại một
   Pod: **Pod không phải một tiến trình mà là môi trường để chạy container**, và nó tồn tại cho
   đến khi bị xóa. Pod **ở lại trên node đã được lập lịch** cho tới khi chạy xong, bị xóa, bị
   trục xuất vì thiếu tài nguyên, hoặc node gặp sự cố — nó không tự chuyển sang node khác.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
