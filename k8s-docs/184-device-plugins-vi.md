# Device Plugin

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/>
>
> Device plugin cho phép bạn cấu hình cluster của mình để hỗ trợ các thiết bị hoặc resource cần thiết lập riêng theo nhà cung cấp, chẳng hạn như GPU, NIC, FPGA hoặc bộ nhớ chính không mất dữ liệu (non-volatile main memory).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 7/7 ·
Kiểm chứng ở Lab 14 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**.

Đây là **cách cũ để expose GPU và thiết bị chuyên dụng**, đối chiếu với DRA mà bạn đã đọc ở
[bài 149](149-dynamic-resource-allocation-vi.md) của giai đoạn 13. Hai cách cùng tồn tại: device
plugin vẫn là cơ chế đang chạy trong hầu hết cluster có GPU hiện nay, còn DRA là hướng thay thế.
Đọc bài này để nói được **hai cách khác nhau ở chỗ nào** — đúng thứ checkpoint giai đoạn 13 hỏi.

Bài dài, nhưng **hơn một nửa là định nghĩa gRPC/protobuf** dành cho người viết plugin. Với vai
trò admin, phần cần hiểu chỉ khoảng ba mục đầu.

**Phải hiểu ở lần đọc này:**

- Luồng đăng ký: plugin tự đăng ký với **kubelet** qua Unix socket
  `/var/lib/kubelet/device-plugins/kubelet.sock`, gửi tên socket, phiên bản API và
  **`ResourceName` dạng `vendor-domain/resourcetype`**. Sau đó **kubelet** mới là bên công bố
  resource này lên API server, như một phần của việc cập nhật trạng thái node.
- Pod xin thiết bị bằng **extended resource** trong `resources.limits`, với hai ràng buộc:
  **chỉ là số nguyên và không thể overcommit**, và **thiết bị không chia sẻ được giữa các
  container**.
- Thứ tự bắt buộc trong *Hiện thực device plugin*: plugin **PHẢI phục vụ dịch vụ gRPC trước** rồi
  mới tự đăng ký. Và khi kubelet khởi động lại, nó **xóa toàn bộ Unix socket** dưới thư mục đó,
  nên plugin phải phát hiện và **tự đăng ký lại**.
- Thiết bị hỏng: kubelet **giảm allocatable** của resource đó trên Node, còn **capacity giữ
  nguyên**. Pod đã được gán thiết bị hỏng **vẫn giữ nguyên thiết bị đó** và sẽ vào pha Failed
  hoặc rơi vào vòng lặp crash tùy `restartPolicy`.
- Triển khai dạng DaemonSet: cần **security context privileged** và phải **mount
  `/var/lib/kubelet/device-plugins` như một volume**, vì đường dẫn này được hard-code trong
  kubelet. Đổi lại, Kubernetes lo việc đặt Pod lên các Node, khởi động lại sau lỗi và nâng cấp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ định nghĩa gRPC và message protobuf (`DevicePlugin`, `ListPodResources…`) | là hợp đồng lập trình cho nhà cung cấp | Lab 14 |
| *Giám sát resource của device plugin* — ba endpoint `List`, `GetAllocatableResources`, `Get` | dành cho agent giám sát, không phải để chạy workload | giai đoạn 11 — bài [162](162-observability-vi.md) |
| *Tích hợp device plugin với Topology Manager*, `TopologyInfo`, NUMA | Topology Manager thuộc nhóm trình quản lý tài nguyên của kubelet | giai đoạn 7 — bài [74](74-resource-managers-vi.md) |
| *Tương thích API* — quy tắc nâng cấp phiên bản device plugin API | chỉ chạm tới khi nâng cấp cluster đang có GPU | Lab 14 |
| Danh sách *Ví dụ về device plugin* của các nhà cung cấp | tra khi có phần cứng thật | Lab 14 |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Kubernetes cung cấp một framework device plugin mà bạn có thể dùng để công bố (advertise) các
resource phần cứng của hệ thống tới kubelet.

Thay vì tùy biến mã nguồn của chính Kubernetes, các nhà cung cấp (vendor) có thể hiện thực một
device plugin mà bạn triển khai thủ công hoặc dưới dạng một DaemonSet.
Các thiết bị được nhắm tới bao gồm GPU, NIC hiệu năng cao, FPGA, adapter InfiniBand, và các
resource tính toán tương tự khác vốn có thể cần khởi tạo và thiết lập riêng theo nhà cung cấp.

## Đăng ký device plugin (Device plugin registration)

Kubelet expose một dịch vụ gRPC `Registration`:

```gRPC
service Registration {
	rpc Register(RegisterRequest) returns (Empty) {}
}
```

Một device plugin có thể tự đăng ký với kubelet thông qua dịch vụ gRPC này.
Trong quá trình đăng ký, device plugin cần gửi:

* Tên Unix socket của nó.
* Phiên bản Device Plugin API mà nó được build dựa trên.
* `ResourceName` mà nó muốn công bố. Ở đây `ResourceName` cần tuân theo
  [quy ước đặt tên extended resource](110-manage-resources-containers-vi.md#extended-resources)
  dạng `vendor-domain/resourcetype`.
  (Ví dụ, một GPU NVIDIA được công bố là `nvidia.com/gpu`.)

Sau khi đăng ký thành công, device plugin gửi cho kubelet danh sách các thiết bị mà nó quản lý,
và kubelet khi đó chịu trách nhiệm công bố những resource này tới API server như một phần của
việc cập nhật trạng thái node của kubelet.
Ví dụ, sau khi một device plugin đăng ký `hardware-vendor.example/foo` với kubelet và báo cáo có
hai thiết bị khỏe mạnh trên một node, trạng thái node được cập nhật để công bố rằng node đó có
2 thiết bị "Foo" đã được cài đặt và sẵn sàng.

Sau đó, người dùng có thể yêu cầu các thiết bị này như một phần của đặc tả Pod
(xem [`container`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)).
Việc yêu cầu extended resource tương tự như cách bạn quản lý request và limit cho các resource
khác, với những khác biệt sau:
* Extended resource chỉ được hỗ trợ dưới dạng resource số nguyên và không thể overcommit.
* Các thiết bị không thể được chia sẻ giữa các container.

### Ví dụ (Example) {#example-pod}

Giả sử một cluster Kubernetes đang chạy một device plugin công bố resource
`hardware-vendor.example/foo` trên một số node nhất định. Dưới đây là ví dụ về một pod yêu cầu
resource này để chạy một workload demo:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: demo-container-1
      image: registry.k8s.io/pause:3.8
      resources:
        limits:
          hardware-vendor.example/foo: 2
#
# Pod này cần 2 thiết bị hardware-vendor.example/foo
# và chỉ có thể được lập lịch lên một Node có khả năng đáp ứng
# nhu cầu đó.
#
# Nếu Node có nhiều hơn 2 thiết bị loại đó đang khả dụng, phần
# còn lại sẽ sẵn sàng cho các Pod khác sử dụng.
```

## Hiện thực device plugin (Device plugin implementation)

Luồng làm việc tổng quát của một device plugin gồm các bước sau:

1. Khởi tạo. Trong giai đoạn này, device plugin thực hiện việc khởi tạo và thiết lập riêng theo
   nhà cung cấp để đảm bảo các thiết bị ở trạng thái sẵn sàng.

1. Plugin khởi động một dịch vụ gRPC, với một Unix socket nằm dưới đường dẫn trên host
   `/var/lib/kubelet/device-plugins/` (đường dẫn này được hard-code và không bị ảnh hưởng bởi
   `--root-dir` của kubelet hay bất kỳ cấu hình nào khác), hiện thực các interface sau:

   ```gRPC
   service DevicePlugin {
         // GetDevicePluginOptions returns options to be communicated with Device Manager.
         rpc GetDevicePluginOptions(Empty) returns (DevicePluginOptions) {}

         // ListAndWatch returns a stream of List of Devices
         // Whenever a Device state change or a Device disappears, ListAndWatch
         // returns the new list
         rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}

         // Allocate is called during container creation so that the Device
         // Plugin can run device specific operations and instruct Kubelet
         // of the steps to make the Device available in the container
         rpc Allocate(AllocateRequest) returns (AllocateResponse) {}

         // GetPreferredAllocation returns a preferred set of devices to allocate
         // from a list of available ones. The resulting preferred allocation is not
         // guaranteed to be the allocation ultimately performed by the
         // devicemanager. It is only designed to help the devicemanager make a more
         // informed allocation decision when possible.
         rpc GetPreferredAllocation(PreferredAllocationRequest) returns (PreferredAllocationResponse) {}

         // PreStartContainer is called, if indicated by Device Plugin during registration phase,
         // before each container start. Device plugin can run device specific operations
         // such as resetting the device before making devices available to the container.
         rpc PreStartContainer(PreStartContainerRequest) returns (PreStartContainerResponse) {}
   }
   ```

   > **Ghi chú:** Các plugin không bắt buộc phải cung cấp hiện thực hữu ích cho
   > `GetPreferredAllocation()` hoặc `PreStartContainer()`. Nếu có, các cờ (flag) cho biết
   > sự sẵn có của những lời gọi này cần được thiết lập trong message `DevicePluginOptions`
   > gửi trả về từ lời gọi `GetDevicePluginOptions()`. `kubelet` sẽ luôn gọi
   > `GetDevicePluginOptions()` để xem những hàm tùy chọn nào khả dụng, trước khi gọi trực tiếp
   > bất kỳ hàm nào trong số đó.

1. Plugin tự đăng ký với kubelet thông qua Unix socket tại đường dẫn trên host
   `/var/lib/kubelet/device-plugins/kubelet.sock`.

   > **Ghi chú:** Thứ tự của luồng làm việc là quan trọng. Một plugin BẮT BUỘC phải bắt đầu phục
   > vụ dịch vụ gRPC trước khi tự đăng ký với kubelet thì việc đăng ký mới thành công.

1. Sau khi tự đăng ký thành công, device plugin chạy ở chế độ phục vụ (serving mode), trong đó nó
   liên tục theo dõi tình trạng sức khỏe của thiết bị và báo cáo lại cho kubelet mỗi khi trạng
   thái thiết bị thay đổi.
   Nó cũng chịu trách nhiệm phục vụ các request gRPC `Allocate`. Trong quá trình `Allocate`,
   device plugin có thể thực hiện các bước chuẩn bị riêng cho thiết bị; ví dụ, dọn dẹp GPU hoặc
   khởi tạo QRNG.
   Nếu các thao tác thành công, device plugin trả về một `AllocateResponse` chứa cấu hình
   container runtime để truy cập các thiết bị đã được cấp phát. Kubelet chuyển thông tin này cho
   container runtime.

   Một `AllocateResponse` chứa không hoặc nhiều đối tượng `ContainerAllocateResponse`. Trong đó,
   device plugin định nghĩa những thay đổi cần được thực hiện đối với định nghĩa của container để
   cung cấp quyền truy cập tới thiết bị. Những thay đổi này bao gồm:

   * [Annotation](20-annotations-vi.md)
   * device node
   * biến môi trường
   * mount
   * tên thiết bị CDI đầy đủ (fully-qualified CDI device name)

   > **Ghi chú:** Việc Device Manager xử lý các tên thiết bị CDI đầy đủ yêu cầu
   > [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
   > `DevicePluginCDIDevices` được bật cho cả kubelet và kube-apiserver. Tính năng này được thêm
   > vào dưới dạng alpha trong Kubernetes v1.28, lên beta ở v1.29 và lên GA ở v1.31.

### Xử lý việc kubelet khởi động lại (Handling kubelet restarts)

Một device plugin được kỳ vọng sẽ phát hiện việc kubelet khởi động lại và tự đăng ký lại với
instance kubelet mới. Một instance kubelet mới sẽ xóa toàn bộ các Unix socket hiện có dưới
`/var/lib/kubelet/device-plugins` (đường dẫn hard-code cho device plugin) khi nó khởi động. Một
device plugin có thể theo dõi việc Unix socket của nó bị xóa và tự đăng ký lại khi sự kiện đó xảy
ra.

### Device plugin và các thiết bị không khỏe mạnh (Device plugin and unhealthy devices)

Có những trường hợp thiết bị bị lỗi hoặc bị tắt. Trách nhiệm của Device Plugin trong trường hợp
này là thông báo cho kubelet về tình huống đó bằng API `ListAndWatchResponse`.

Khi một thiết bị bị đánh dấu là không khỏe mạnh, kubelet sẽ giảm số lượng allocatable của
resource này trên Node để phản ánh còn bao nhiêu thiết bị có thể dùng để lập lịch cho các pod
mới. Số lượng capacity của resource sẽ không thay đổi.

Những Pod đã được gán cho các thiết bị bị lỗi sẽ vẫn tiếp tục được gán cho thiết bị đó. Thông
thường, mã chạy dựa trên thiết bị sẽ bắt đầu gặp lỗi và Pod có thể rơi vào pha Failed nếu
`restartPolicy` của Pod không phải là `Always`, hoặc rơi vào vòng lặp crash trong trường hợp
ngược lại.

Trước Kubernetes v1.31, cách để biết một Pod có liên quan tới thiết bị bị lỗi hay không là dùng
[PodResources API](#monitoring-device-plugin-resources).

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]` (bật mặc định)

Khi feature gate `ResourceHealthStatus` được bật (ở mức beta và được bật mặc định từ v1.36),
trường `allocatedResourcesStatus` được thêm vào trạng thái của mỗi container, bên trong phần
`.status` của mỗi Pod. Trường `allocatedResourcesStatus` báo cáo thông tin sức khỏe cho từng
thiết bị được gán cho container.
Mỗi mục thông tin sức khỏe của resource có thể bao gồm một trường `message` tùy chọn với ngữ
cảnh bổ sung dễ đọc cho con người về tình trạng sức khỏe, chẳng hạn như chi tiết lỗi hoặc nguyên
nhân thất bại.

Với một Pod bị lỗi, hoặc khi bạn nghi ngờ có sự cố, bạn có thể dùng trạng thái này để hiểu liệu
hành vi của Pod có liên quan tới lỗi thiết bị hay không. Ví dụ, nếu một accelerator đang báo cáo
sự kiện quá nhiệt, trường `allocatedResourcesStatus` có thể báo cáo điều này.

## Triển khai device plugin (Device plugin deployment)

Bạn có thể triển khai một device plugin dưới dạng DaemonSet, dưới dạng một package cho hệ điều
hành của node, hoặc thủ công.

Thư mục chuẩn `/var/lib/kubelet/device-plugins` (được hard-code trong kubelet) yêu cầu quyền truy
cập privileged, vì vậy device plugin phải chạy trong một security context privileged.
Nếu bạn triển khai device plugin dưới dạng DaemonSet, `/var/lib/kubelet/device-plugins` phải được
mount như một volume trong
[PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
của plugin.

Nếu bạn chọn cách tiếp cận DaemonSet, bạn có thể dựa vào Kubernetes để: đặt Pod của device plugin
lên các Node, khởi động lại Pod daemon sau khi gặp lỗi, và giúp tự động hóa việc nâng cấp.

## Tương thích API (API compatibility)

Trước đây, sơ đồ đánh phiên bản yêu cầu phiên bản API của Device Plugin phải khớp chính xác với
phiên bản của Kubelet. Kể từ khi tính năng này lên Beta ở v1.12, đây không còn là một yêu cầu bắt
buộc nữa. API đã được đánh phiên bản và ổn định kể từ khi tính năng này lên Beta. Vì thế, việc
nâng cấp kubelet đáng lẽ diễn ra suôn sẻ, nhưng vẫn có thể có những thay đổi trong API trước khi
nó ổn định hoàn toàn, nên không đảm bảo rằng việc nâng cấp sẽ không gây gián đoạn.

> **Ghi chú:** Mặc dù thành phần Device Manager của Kubernetes là một tính năng đã khả dụng rộng
> rãi (generally available), _device plugin API_ thì chưa ổn định. Để biết thông tin về device
> plugin API và tính tương thích giữa các phiên bản, hãy đọc
> [Device Plugin API versions](https://kubernetes.io/docs/reference/node/device-plugin-api-versions/).

Với tư cách là một dự án, Kubernetes khuyến nghị các nhà phát triển device plugin:

* Theo dõi các thay đổi của Device Plugin API trong những bản phát hành tương lai.
* Hỗ trợ nhiều phiên bản của device plugin API để tương thích ngược/tương thích xuôi.

Để chạy device plugin trên các node cần được nâng cấp lên một bản Kubernetes có phiên bản device
plugin API mới hơn, hãy nâng cấp device plugin của bạn để hỗ trợ cả hai phiên bản trước khi nâng
cấp những node này. Cách tiếp cận đó sẽ đảm bảo việc cấp phát thiết bị hoạt động liên tục trong
suốt quá trình nâng cấp.

## Giám sát resource của device plugin (Monitoring device plugin resources) {#monitoring-device-plugin-resources}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

Để giám sát các resource do device plugin cung cấp, các agent giám sát cần có khả năng khám phá
tập hợp thiết bị đang được sử dụng trên node và lấy được metadata mô tả metric đó nên được gắn
với container nào. Các metric [Prometheus](https://prometheus.io/) do các agent giám sát thiết bị
expose nên tuân theo
[Kubernetes Instrumentation Guidelines](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-instrumentation/metric-instrumentation.md),
định danh container bằng các label prometheus `pod`, `namespace` và `container`.

Kubelet cung cấp một dịch vụ gRPC để cho phép khám phá các thiết bị đang được sử dụng, và để cung
cấp metadata cho những thiết bị này:

```gRPC
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Get(GetPodResourcesRequest) returns (GetPodResourcesResponse) {}
}
```

### Endpoint gRPC `List` {#grpc-endpoint-list}

Endpoint `List` cung cấp thông tin về resource của các pod đang chạy, với các chi tiết như id của
những CPU được cấp phát độc quyền, id thiết bị theo báo cáo của device plugin và id của NUMA node
nơi các thiết bị này được cấp phát. Ngoài ra, với các máy dựa trên NUMA, nó chứa thông tin về bộ
nhớ và hugepage được dành riêng cho một container.

Kể từ Kubernetes v1.27, endpoint `List` có thể cung cấp thông tin về resource của các pod đang
chạy được cấp phát trong các `ResourceClaims` bởi API `DynamicResourceAllocation`.
Kể từ Kubernetes v1.34, tính năng này được bật mặc định.

```gRPC
// ListPodResourcesResponse is the response returned by List function
message ListPodResourcesResponse {
    repeated PodResources pod_resources = 1;
}

// PodResources contains information about the node resources assigned to a pod
message PodResources {
    string name = 1;
    string namespace = 2;
    repeated ContainerResources containers = 3;
}

// ContainerResources contains information about the resources assigned to a container
message ContainerResources {
    string name = 1;
    repeated ContainerDevices devices = 2;
    repeated int64 cpu_ids = 3;
    repeated ContainerMemory memory = 4;
    repeated DynamicResource dynamic_resources = 5;
}

// ContainerMemory contains information about memory and hugepages assigned to a container
message ContainerMemory {
    string memory_type = 1;
    uint64 size = 2;
    TopologyInfo topology = 3;
}

// Topology describes hardware topology of the resource
message TopologyInfo {
        repeated NUMANode nodes = 1;
}

// NUMA representation of NUMA node
message NUMANode {
        int64 ID = 1;
}

// ContainerDevices contains information about the devices assigned to a container
message ContainerDevices {
    string resource_name = 1;
    repeated string device_ids = 2;
    TopologyInfo topology = 3;
}

// DynamicResource contains information about the devices assigned to a container by Dynamic Resource Allocation
message DynamicResource {
    string class_name = 1;
    string claim_name = 2;
    string claim_namespace = 3;
    repeated ClaimResource claim_resources = 4;
}

// ClaimResource contains per-plugin resource information
message ClaimResource {
    repeated CDIDevice cdi_devices = 1 [(gogoproto.customname) = "CDIDevices"];
}

// CDIDevice specifies a CDI device information
message CDIDevice {
    // Fully qualified CDI device name
    // for example: vendor.com/gpu=gpudevice1
    // see more details in the CDI specification:
    // https://github.com/container-orchestrated-devices/container-device-interface/blob/main/SPEC.md
    string name = 1;
}
```

> **Ghi chú:** cpu_ids trong `ContainerResources` ở endpoint `List` tương ứng với các CPU được cấp
> phát độc quyền cho một container cụ thể. Nếu mục tiêu là đánh giá các CPU thuộc pool dùng chung,
> endpoint `List` cần được dùng kết hợp với endpoint `GetAllocatableResources` như giải thích
> dưới đây:
> 1. Gọi `GetAllocatableResources` để lấy danh sách tất cả các CPU allocatable
> 2. Gọi `GetCpuIds` trên tất cả `ContainerResources` trong hệ thống
> 3. Trừ đi tất cả các CPU thu được từ các lời gọi `GetCpuIds` khỏi kết quả của lời gọi `GetAllocatableResources`

### Endpoint gRPC `GetAllocatableResources` {#grpc-endpoint-getallocatableresources}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

GetAllocatableResources cung cấp thông tin về các resource ban đầu khả dụng trên worker node.
Nó cung cấp nhiều thông tin hơn so với những gì kubelet xuất ra cho APIServer.

> **Ghi chú:** `GetAllocatableResources` chỉ nên được dùng để đánh giá các resource
> [allocatable](253-reserve-compute-resources-vi.md#node-allocatable)
> trên một node. Nếu mục tiêu là đánh giá các resource còn trống/chưa được cấp phát thì nó cần
> được dùng kết hợp với endpoint List(). Kết quả thu được từ `GetAllocatableResources` sẽ giữ
> nguyên trừ khi các resource nền tảng expose cho kubelet thay đổi. Điều này hiếm khi xảy ra
> nhưng khi nó xảy ra (ví dụ: hotplug/hotunplug, tình trạng sức khỏe thiết bị thay đổi), client
> được kỳ vọng sẽ gọi endpoint `GetAllocatableResources`.
>
> Tuy nhiên, chỉ gọi endpoint `GetAllocatableResources` là chưa đủ trong trường hợp cpu và/hoặc
> memory được cập nhật, và Kubelet cần được khởi động lại để phản ánh đúng capacity và allocatable
> của resource.

```gRPC
// AllocatableResourcesResponses contains information about all the devices known by the kubelet
message AllocatableResourcesResponse {
    repeated ContainerDevices devices = 1;
    repeated int64 cpu_ids = 2;
    repeated ContainerMemory memory = 3;
}
```

`ContainerDevices` có expose thông tin topology, khai báo thiết bị này có ái lực (affine) với các
NUMA cell nào. Các NUMA cell được định danh bằng một ID số nguyên dạng opaque, mà giá trị của nó
nhất quán với những gì device plugin báo cáo
[khi chúng tự đăng ký với kubelet](#device-plugin-integration-with-the-topology-manager).

Dịch vụ gRPC này được phục vụ qua một unix socket tại `pod-resources/kubelet.sock` bên trong thư
mục gốc của kubelet (thường là `/var/lib/kubelet/pod-resources/kubelet.sock`).
Các agent giám sát cho resource của device plugin có thể được triển khai dưới dạng một daemon,
hoặc dưới dạng một DaemonSet. Thư mục chuẩn `pod-resources` bên trong thư mục gốc của kubelet
(thường là `/var/lib/kubelet/pod-resources`) yêu cầu quyền truy cập privileged, vì vậy các agent
giám sát phải chạy trong một security context privileged. Nếu một agent giám sát thiết bị chạy
dưới dạng DaemonSet, thư mục `pod-resources` phải được mount như một volume trong
[PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
của agent giám sát thiết bị đó.

> **Ghi chú:** Khi truy cập `pod-resources/kubelet.sock` từ một DaemonSet hoặc bất kỳ ứng dụng nào
> khác được triển khai dưới dạng container trên host, mà ứng dụng đó mount socket như một volume,
> một thực hành tốt là mount thư mục `pod-resources` thay vì chính file socket. Điều này sẽ đảm
> bảo rằng sau khi kubelet khởi động lại, container vẫn có thể kết nối lại tới socket này.
>
> Trên một node Linux điển hình, điều này có nghĩa là mount `/var/lib/kubelet/pod-resources/`
> thay vì `/var/lib/kubelet/pod-resources/kubelet.sock`.
>
> Các mount của container được quản lý bằng inode tham chiếu tới socket hoặc thư mục, tùy vào cái
> nào được mount. Khi kubelet khởi động lại, socket bị xóa và một socket mới được tạo ra, trong
> khi thư mục vẫn không bị đụng tới. Vì vậy inode gốc của socket trở nên không dùng được nữa.
> Còn inode của thư mục thì sẽ tiếp tục hoạt động.

### Endpoint gRPC `Get` {#grpc-endpoint-get}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

Endpoint `Get` cung cấp thông tin về resource của một Pod đang chạy. Nó expose thông tin tương tự
những gì được mô tả ở endpoint `List`. Endpoint `Get` yêu cầu `PodName` và `PodNamespace` của Pod
đang chạy.

```gRPC
// GetPodResourcesRequest contains information about the pod
message GetPodResourcesRequest {
    string pod_name = 1;
    string pod_namespace = 2;
}
```

Endpoint `Get` có thể cung cấp thông tin Pod liên quan tới các dynamic resource được cấp phát bởi
API dynamic resource allocation.
Kể từ Kubernetes v1.34, tính năng này được bật mặc định.

## Tích hợp device plugin với Topology Manager (Device plugin integration with the Topology Manager) {#device-plugin-integration-with-the-topology-manager}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Topology Manager là một thành phần của Kubelet cho phép các resource được phối hợp theo cách căn
chỉnh với Topology. Để làm được điều này, Device Plugin API đã được mở rộng để bao gồm một struct
`TopologyInfo`.

```gRPC
message TopologyInfo {
    repeated NUMANode nodes = 1;
}

message NUMANode {
    int64 ID = 1;
}
```

Những Device Plugin muốn tận dụng Topology Manager có thể gửi trả về một struct TopologyInfo đã
được điền dữ liệu như một phần của việc đăng ký thiết bị, cùng với các ID thiết bị và tình trạng
sức khỏe của thiết bị. Device manager sau đó sẽ dùng thông tin này để tham vấn Topology Manager và
đưa ra các quyết định gán resource.

`TopologyInfo` hỗ trợ đặt trường `nodes` thành `nil` hoặc thành một danh sách các NUMA node. Điều
này cho phép Device Plugin công bố một thiết bị trải rộng trên nhiều NUMA node.

Đặt `TopologyInfo` thành `nil` hoặc cung cấp một danh sách NUMA node rỗng cho một thiết bị nhất
định có nghĩa là Device Plugin không có ưu tiên về ái lực NUMA (NUMA affinity) đối với thiết bị đó.

Một ví dụ về struct `TopologyInfo` được Device Plugin điền dữ liệu cho một thiết bị:

```
pluginapi.Device{ID: "25102017", Health: pluginapi.Healthy, Topology:&pluginapi.TopologyInfo{Nodes: []*pluginapi.NUMANode{&pluginapi.NUMANode{ID: 0,},}}}
```

## Ví dụ về device plugin (Device plugin examples) {#examples}

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Dưới đây là một số ví dụ về các hiện thực device plugin:

* [Akri](https://github.com/project-akri/akri), cho phép bạn dễ dàng expose các thiết bị lá (leaf device) không đồng nhất (chẳng hạn như camera IP và thiết bị USB).
* [AMD GPU device plugin](https://github.com/ROCm/k8s-device-plugin)
* [generic device plugin](https://github.com/squat/generic-device-plugin) cho các thiết bị Linux thông dụng và thiết bị USB
* [HAMi](https://github.com/Project-HAMi/HAMi) — middleware ảo hóa cho tính toán AI không đồng nhất (ví dụ: NVIDIA, Cambricon, Hygon, Iluvatar, MThreads, Ascend, Metax)
* [Các Intel device plugin](https://github.com/intel/intel-device-plugins-for-kubernetes) cho các
  thiết bị Intel GPU, FPGA, QAT, VPU, SGX, DSA, DLB và IAA
* [Các KubeVirt device plugin](https://github.com/kubevirt/kubernetes-device-plugins) cho ảo hóa
  có hỗ trợ phần cứng
* [NVIDIA GPU device plugin](https://github.com/NVIDIA/k8s-device-plugin), device plugin chính
  thức của NVIDIA để expose GPU NVIDIA và giám sát sức khỏe GPU
* [NVIDIA GPU device plugin cho Container-Optimized OS](https://github.com/GoogleCloudPlatform/container-engine-accelerators/tree/master/cmd/nvidia_gpu)
* [RDMA device plugin](https://github.com/hustcat/k8s-rdma-device-plugin)
* [SocketCAN device plugin](https://github.com/collabora/k8s-socketcan)
* [Solarflare device plugin](https://github.com/vikaschoudhary16/sfc-device-plugin)
* [SR-IOV Network device plugin](https://github.com/intel/sriov-network-device-plugin)
* [Các Xilinx FPGA device plugin](https://github.com/Xilinx/FPGA_as_a_Service/tree/master/k8s-device-plugin) cho các thiết bị Xilinx FPGA

## Tiếp theo (What's next)

* Tìm hiểu về [lập lịch resource GPU](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
  bằng device plugin
* Tìm hiểu về [việc công bố extended resource](209-extended-resource-node-vi.md)
  trên một node
* Tìm hiểu về [Topology Manager](259-topology-manager-vi.md)
* Đọc về việc dùng [tăng tốc phần cứng cho TLS ingress](https://kubernetes.io/blog/2019/04/24/hardware-accelerated-ssl/tls-termination-in-ingress-controllers-using-kubernetes-device-plugins-and-runtimeclass/)
  với Kubernetes
* Đọc thêm về [Cấp phát Extended Resource bằng DRA](149-dynamic-resource-allocation-vi.md#extended-resource)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Câu bẫy phân biệt: với device plugin, Pod xin thiết bị bằng **thứ gì**, và hai ràng buộc nào
   áp lên con số đó? So với DRA ở bài [149](149-dynamic-resource-allocation-vi.md), vì sao cách
   này không diễn đạt được yêu cầu kiểu "cho tôi một thiết bị có thuộc tính X"?
2. Một thiết bị trên `k8s-worker2` hỏng và device plugin báo unhealthy. Trong `kubectl describe
   node k8s-worker2`, con số nào thay đổi và con số nào **không** đổi? Pod đang giữ đúng thiết bị
   đó bị gì?
3. Bạn viết một device plugin và cho nó đăng ký với kubelet ngay khi khởi động, rồi mới mở dịch
   vụ gRPC. Bài nói gì về thứ tự này?
4. kubelet trên `k8s-worker2` được restart. Device plugin phải làm gì, và nhờ dấu hiệu nào nó
   biết cần làm?
5. Bạn triển khai device plugin dạng DaemonSet. Hai điều kiện bắt buộc bài nêu là gì, và bạn
   được lợi gì khi chọn DaemonSet thay vì cài thủ công?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod xin bằng một **extended resource** đặt trong `resources.limits`, tên theo quy ước
   `vendor-domain/resourcetype` (ví dụ `hardware-vendor.example/foo: 2`). Hai ràng buộc:
   **chỉ là resource số nguyên và không thể overcommit**, và **thiết bị không chia sẻ được giữa
   các container**. Vì thứ duy nhất Pod nói được là **một cái tên và một con số đếm**, mô hình
   này **không có chỗ để diễn đạt thuộc tính** — mọi thiết bị mang cùng `ResourceName` bị coi là
   thay thế được cho nhau. Đó chính là ranh giới mà DRA ở bài
   [149](149-dynamic-resource-allocation-vi.md) vượt qua bằng ResourceClaim và DeviceClass. Chỗ
   dễ nhầm: device plugin **chưa hề bị thay thế** — bài này vẫn là `stable` và trong chính bài,
   endpoint `List` phải bổ sung trường `DynamicResource` để báo cáo cả thiết bị do DRA cấp phát,
   cho thấy hai cơ chế cùng tồn tại trên một node.
2. **Allocatable giảm, capacity giữ nguyên.** Bài viết: "kubelet sẽ giảm số lượng allocatable của
   resource này trên Node để phản ánh còn bao nhiêu thiết bị có thể dùng để lập lịch cho các pod
   mới. Số lượng capacity của resource sẽ không thay đổi." Pod đang giữ thiết bị hỏng **vẫn tiếp
   tục được gán cho thiết bị đó** — Kubernetes không tự dời nó đi; mã chạy trên thiết bị sẽ bắt
   đầu lỗi và Pod **rơi vào pha Failed nếu `restartPolicy` không phải `Always`**, ngược lại thì
   vào **vòng lặp crash**. Từ v1.36, feature gate `ResourceHealthStatus` bật mặc định thêm
   `allocatedResourcesStatus` vào `.status` của Pod để bạn biết Pod hỏng có phải do thiết bị.
3. **Sai thứ tự.** Bài ghi rõ trong ghi chú: "Thứ tự của luồng làm việc là quan trọng. Một plugin
   **BẮT BUỘC** phải bắt đầu phục vụ dịch vụ gRPC **trước khi** tự đăng ký với kubelet thì việc
   đăng ký mới thành công." Đăng ký trước nghĩa là kubelet có thể gọi tới một socket chưa ai
   phục vụ.
4. Plugin phải **tự đăng ký lại với instance kubelet mới**. Dấu hiệu: một kubelet mới khởi động
   sẽ **xóa toàn bộ các Unix socket hiện có** dưới `/var/lib/kubelet/device-plugins`, nên plugin
   chỉ cần **theo dõi việc Unix socket của chính nó bị xóa** và đăng ký lại khi sự kiện đó xảy
   ra.
5. Hai điều kiện: plugin phải chạy trong một **security context privileged** (vì thư mục chuẩn
   `/var/lib/kubelet/device-plugins` được hard-code trong kubelet và yêu cầu quyền privileged),
   và **`/var/lib/kubelet/device-plugins` phải được mount như một volume** trong PodSpec của
   plugin. Lợi ích của DaemonSet: bạn **dựa vào Kubernetes** để đặt Pod của plugin lên các Node,
   khởi động lại Pod daemon sau khi gặp lỗi, và tự động hóa việc nâng cấp.

</details>

Đây là bài cuối của **Giai đoạn 14**. Trả lời được hết bảy bài thì bạn sẵn sàng vào Lab 14 (chưa
viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)); trước đó hãy chốt lại checkpoint của giai
đoạn trong [lộ trình](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng).
