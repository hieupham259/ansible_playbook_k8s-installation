# Lưu trữ tạm thời cục bộ (Local ephemeral storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/ephemeral-storage/>

Các node có lưu trữ tạm thời cục bộ (local ephemeral storage), được hỗ trợ bởi
các thiết bị ghi được gắn cục bộ hoặc, đôi khi, bởi RAM.
"Tạm thời" (ephemeral) nghĩa là không có bảo đảm dài hạn nào về độ bền của dữ liệu.

Các Pod dùng lưu trữ tạm thời cục bộ làm không gian nháp (scratch space), làm cache, và để chứa log.
Kubelet có thể cung cấp không gian nháp cho các Pod bằng cách dùng lưu trữ tạm thời cục bộ để
mount các volume [`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
vào các container.

Kubelet cũng dùng loại lưu trữ này để chứa
[log container cấp node](https://kubernetes.io/docs/concepts/cluster-administration/logging/#logging-at-the-node-level),
các container image, và các lớp ghi được (writable layer) của các container đang chạy.

> **Thận trọng:**
> Nếu một node gặp sự cố, dữ liệu trong lưu trữ tạm thời của node đó có thể bị mất.
> Ứng dụng của bạn không thể kỳ vọng bất kỳ SLA hiệu năng nào (ví dụ IOPS của đĩa)
> từ lưu trữ tạm thời cục bộ.

> **Ghi chú:**
> Để resource quota hoạt động với ephemeral-storage, cần làm hai việc:
>
> * Quản trị viên đặt resource quota cho ephemeral-storage trong một namespace.
> * Người dùng cần chỉ định limit cho tài nguyên ephemeral-storage trong spec của Pod.
>
> Nếu người dùng không chỉ định limit tài nguyên ephemeral-storage trong spec của Pod,
> resource quota sẽ không được áp dụng cho ephemeral-storage.

Kubernetes cho phép bạn theo dõi, dự trữ và giới hạn lượng
lưu trữ tạm thời cục bộ mà một Pod có thể tiêu thụ.

## Các cấu hình cho lưu trữ tạm thời cục bộ (Configurations for local ephemeral storage) {#configurations}

Kubernetes hỗ trợ các cách sau để cấu hình lưu trữ tạm thời cục bộ trên một
node:

#### Một filesystem duy nhất (Single filesystem)

Trong cấu hình này, bạn đặt tất cả các loại dữ liệu tạm thời cục bộ khác nhau
(các volume `emptyDir`, các lớp ghi được, container image, log) vào một filesystem duy nhất.

Kubelet cũng ghi
[log container cấp node](https://kubernetes.io/docs/concepts/cluster-administration/logging/#logging-at-the-node-level)
và xử lý chúng tương tự như lưu trữ tạm thời cục bộ.

Kubelet ghi log vào các file bên trong thư mục log đã được cấu hình (`/var/log`
theo mặc định); và có một thư mục cơ sở cho các dữ liệu khác được lưu cục bộ
(`/var/lib/kubelet` theo mặc định).

Thông thường, cả `/var/lib/kubelet` và `/var/log` đều nằm trên filesystem gốc của hệ thống,
và kubelet được thiết kế với bố cục đó trong đầu.

Node của bạn có thể có bao nhiêu filesystem khác không dùng cho Kubernetes
tùy thích.

#### Filesystem của runtime (Runtime filesystem)

Bạn dùng một filesystem trên node cho dữ liệu tạm thời từ các Pod đang chạy, chẳng hạn
log và các volume `emptyDir`. Bạn cũng có thể dùng filesystem này cho dữ liệu khác,
chẳng hạn log hệ thống không liên quan tới Kubernetes; nó thậm chí có thể là
filesystem gốc.

Kubelet cũng ghi
[log container cấp node](https://kubernetes.io/docs/concepts/cluster-administration/logging/#logging-at-the-node-level)
vào filesystem thứ nhất, và xử lý chúng tương tự như lưu trữ tạm thời cục bộ.

Bạn cũng dùng thêm một filesystem riêng, được hỗ trợ bởi một thiết bị lưu trữ logic khác.
Trong cấu hình này, container runtime lưu cả các lớp container image
lẫn các lớp ghi được trên filesystem thứ hai này. Hãy cấu hình vị trí lưu trữ này
trong container runtime của bạn, không phải trong kubelet.

Filesystem thứ nhất không chứa bất kỳ lớp image hay lớp ghi được nào.

Node của bạn có thể có bao nhiêu filesystem khác không dùng cho Kubernetes
tùy thích.

#### Tách filesystem cho image (Split image filesystem)

Trong cấu hình này, các lớp container image nằm trên một filesystem riêng, còn
các lớp ghi được của container nằm trên cùng filesystem với dữ liệu tạm thời
của kubelet, chẳng hạn log và các volume `emptyDir`.

Bố cục này yêu cầu hỗ trợ các tín hiệu trục xuất (eviction signal) `containerfs`. Để biết chi tiết
về feature gate và các container runtime hỗ trợ bố cục này, xem
[trục xuất do áp lực node (node-pressure eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#filesystem-signals).

Trang [node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#filesystem-signals)
gọi các filesystem được quan sát này là `nodefs`, `imagefs`, và
`containerfs`. Những tên này không phải lúc nào cũng có nghĩa là các mount point riêng biệt.

Kubelet có thể đo mức sử dụng lưu trữ cục bộ khi bạn thiết lập node theo một trong
các cấu hình được hỗ trợ cho lưu trữ tạm thời cục bộ.

Nếu bạn dùng một cấu hình khác, kubelet sẽ không áp dụng giới hạn tài nguyên
cho lưu trữ tạm thời cục bộ.

> **Ghi chú:**
> Kubelet theo dõi các volume emptyDir dạng `tmpfs` như là mức sử dụng bộ nhớ của container,
> thay vì như lưu trữ tạm thời cục bộ.

> **Ghi chú:**
> Kubelet chỉ có thể theo dõi lưu trữ tạm thời trên các filesystem mà nó quan sát được
> thông qua các bố cục được hỗ trợ. Nếu bạn mount thêm các filesystem dưới các đường dẫn như
> `/var/lib/kubelet`, `/var/log`, hoặc thư mục lưu trữ của container runtime
> nằm ngoài các bố cục đó, kubelet có thể báo cáo lưu trữ tạm thời không chính xác.

## Thiết lập request và limit cho lưu trữ tạm thời cục bộ (Setting requests and limits for local ephemeral storage) {#requests-limits}

Bạn có thể chỉ định `ephemeral-storage` để quản lý lưu trữ tạm thời cục bộ. Mỗi
container của một Pod có thể chỉ định một hoặc cả hai mục sau:

* `spec.containers[].resources.limits.ephemeral-storage`
* `spec.containers[].resources.requests.ephemeral-storage`

Limit và request cho `ephemeral-storage` được đo bằng số lượng byte.
Bạn có thể biểu diễn dung lượng lưu trữ dưới dạng một số nguyên thuần hoặc một số thập phân cố định
kèm một trong các hậu tố: E, P, T, G, M, k. Bạn cũng có thể dùng các hậu tố tương đương
theo lũy thừa của hai: Ei, Pi, Ti, Gi, Mi, Ki. Ví dụ, các số lượng sau đây đều biểu diễn
giá trị xấp xỉ như nhau:

- `128974848`
- `129e6`
- `129M`
- `123Mi`

Hãy chú ý chữ hoa/chữ thường của các hậu tố. Nếu bạn request `400m` ephemeral-storage, đó là request
cho 0.4 byte. Người gõ như vậy nhiều khả năng muốn xin 400 mebibyte (`400Mi`)
hoặc 400 megabyte (`400M`).

Trong ví dụ sau, Pod có hai container. Mỗi container có request
2GiB lưu trữ tạm thời cục bộ. Mỗi container có limit 4GiB lưu trữ tạm thời
cục bộ. Do đó, Pod có request 4GiB lưu trữ tạm thời cục bộ, và
limit 8GiB lưu trữ tạm thời cục bộ. 500Mi trong limit đó có thể bị
tiêu thụ bởi volume `emptyDir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
    volumeMounts:
    - name: ephemeral
      mountPath: "/tmp"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
    volumeMounts:
    - name: ephemeral
      mountPath: "/tmp"
  volumes:
    - name: ephemeral
      emptyDir:
        sizeLimit: 500Mi
```

## Cách các Pod có request ephemeral-storage được lập lịch (How Pods with ephemeral-storage requests are scheduled)

Khi bạn tạo một Pod, bộ lập lịch (scheduler) của Kubernetes chọn một node để Pod
chạy trên đó. Mỗi node có một lượng lưu trữ tạm thời cục bộ tối đa mà nó có thể cung cấp cho các Pod.
Để biết thêm thông tin, xem
[Node Allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable).

Bộ lập lịch bảo đảm rằng tổng các request tài nguyên của các container được lập lịch nhỏ hơn dung lượng của node.

## Quản lý mức tiêu thụ lưu trữ tạm thời (Ephemeral storage consumption management) {#resource-emphemeralstorage-consumption}

Nếu kubelet đang quản lý lưu trữ tạm thời cục bộ như một tài nguyên, thì
kubelet đo mức sử dụng lưu trữ trong:

- các volume `emptyDir`, ngoại trừ các volume `emptyDir` dạng _tmpfs_
- các thư mục chứa log cấp node
- các lớp container ghi được

Nếu một Pod dùng nhiều lưu trữ tạm thời hơn mức bạn cho phép, kubelet
đặt một tín hiệu trục xuất (eviction signal) kích hoạt việc trục xuất Pod.

Với cách ly cấp container, nếu mức sử dụng của lớp ghi được và log
của một container vượt quá limit lưu trữ của nó, kubelet đánh dấu Pod để trục xuất.

Với cách ly cấp pod, kubelet tính ra limit lưu trữ tổng thể của Pod bằng cách
cộng các limit của các container trong Pod đó. Trong trường hợp này, nếu tổng
mức sử dụng lưu trữ tạm thời cục bộ từ tất cả các container cùng với các volume `emptyDir`
của Pod vượt quá limit lưu trữ tổng thể của Pod, thì kubelet cũng đánh dấu Pod
để trục xuất.

> **Thận trọng:**
> Nếu kubelet không đo lưu trữ tạm thời cục bộ, thì một Pod
> vượt quá limit lưu trữ cục bộ của nó sẽ không bị trục xuất vì vi phạm
> giới hạn tài nguyên lưu trữ cục bộ.
>
> Tuy nhiên, nếu không gian filesystem cho các lớp container ghi được, log cấp node,
> hoặc các volume `emptyDir` xuống thấp, node sẽ tự
> đánh dấu (taint) chính nó là đang thiếu lưu trữ cục bộ,
> và taint này kích hoạt việc trục xuất đối với bất kỳ Pod nào không dung thứ (tolerate) taint đó một cách tường minh.
>
> Xem các [cấu hình](#configurations) được hỗ trợ cho lưu trữ tạm thời cục bộ.

Kubelet hỗ trợ các cách khác nhau để đo mức sử dụng lưu trữ của Pod:

#### Quét định kỳ (Periodic scanning)

Kubelet thực hiện các lần kiểm tra định kỳ theo lịch, quét từng volume `emptyDir`,
thư mục log của container, và lớp container ghi được.

Lần quét đo lượng không gian đã được sử dụng.

> **Ghi chú:**
> Trong chế độ này, kubelet không theo dõi các file descriptor đang mở
> của các file đã bị xóa.
>
> Nếu bạn (hoặc một container) tạo một file bên trong một volume `emptyDir`,
> sau đó có thứ gì đó mở file này, và bạn xóa file trong khi nó vẫn đang mở,
> thì inode của file đã xóa vẫn tồn tại cho đến khi bạn đóng file đó,
> nhưng kubelet không phân loại phần không gian này là đang được sử dụng.

#### Hạn ngạch project của filesystem (Filesystem project quota)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [beta]` (bật mặc định: false)

Hạn ngạch project (project quota) là một tính năng ở cấp hệ điều hành để quản lý
mức sử dụng lưu trữ trên các filesystem. Với Kubernetes, bạn có thể bật hạn ngạch
project để giám sát mức sử dụng lưu trữ. Hãy bảo đảm rằng filesystem
hỗ trợ các volume `emptyDir` trên node có hỗ trợ hạn ngạch project.
Ví dụ, XFS và ext4fs cung cấp hạn ngạch project.

> **Ghi chú:**
> Hạn ngạch project cho phép bạn giám sát mức sử dụng lưu trữ; chúng không cưỡng chế các limit.

Kubernetes dùng các project ID bắt đầu từ `1048576`. Các ID đang được dùng được
đăng ký trong `/etc/projects` và `/etc/projid`. Nếu các project ID trong
khoảng này được dùng cho các mục đích khác trên hệ thống, các project ID
đó phải được đăng ký trong `/etc/projects` và `/etc/projid` để
Kubernetes không dùng đến chúng.

Hạn ngạch nhanh hơn và chính xác hơn so với quét thư mục.
Khi một thư mục được gán vào một project, tất cả các file được tạo dưới thư mục đó
đều được tạo trong project đó, và kernel chỉ cần theo dõi
số block đang được sử dụng bởi các file trong project đó.
Nếu một file được tạo rồi bị xóa, nhưng vẫn còn một file descriptor đang mở,
nó tiếp tục chiếm dụng không gian. Việc theo dõi bằng hạn ngạch ghi nhận chính xác phần không gian đó,
trong khi việc quét thư mục bỏ sót phần lưu trữ được dùng bởi các file đã xóa.

Để dùng hạn ngạch nhằm theo dõi mức sử dụng tài nguyên của một pod, pod đó phải nằm trong
một user namespace. Bên trong các user namespace, kernel hạn chế các thay đổi
đối với projectID trên filesystem, bảo đảm độ tin cậy của các số liệu
lưu trữ được tính bằng hạn ngạch.

Nếu bạn muốn dùng hạn ngạch project, bạn nên:

* Bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  `LocalStorageCapacityIsolationFSQuotaMonitoring=true`
  bằng trường `featureGates` trong
  [cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

* Bảo đảm rằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  `UserNamespacesSupport`
  được bật, và rằng kernel, CRI implementation và OCI runtime hỗ trợ user namespace.

* Bảo đảm rằng filesystem gốc (hoặc filesystem runtime tùy chọn)
  đã được bật hạn ngạch project. Tất cả các filesystem XFS đều hỗ trợ hạn ngạch project.
  Với các filesystem ext4, bạn cần bật tính năng theo dõi hạn ngạch project
  khi filesystem chưa được mount.

  ```bash
  # Với ext4, khi /dev/block-device chưa được mount
  sudo tune2fs -O project -Q prjquota /dev/block-device
  ```

* Bảo đảm rằng filesystem gốc (hoặc filesystem runtime tùy chọn) được
  mount với hạn ngạch project được bật. Với cả XFS lẫn ext4fs, tùy chọn
  mount có tên là `prjquota`.

Nếu bạn không muốn dùng hạn ngạch project, bạn nên:

* Tắt [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  `LocalStorageCapacityIsolationFSQuotaMonitoring`
  bằng trường `featureGates` trong
  [cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

## Tiếp theo (What's next)

* Đọc về [hạn ngạch project](https://www.linux.org/docs/man8/xfs_quota.html) trong XFS
