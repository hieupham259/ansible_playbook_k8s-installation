# Truy cập metadata thiết bị DRA (Access DRA Device Metadata)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/
>
> Trang này hướng dẫn cách truy cập metadata của thiết bị từ các container sử dụng cấp phát
> tài nguyên động (dynamic resource allocation — DRA), bằng cách đọc các file JSON tại những
> đường dẫn quy ước bên trong container.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Trang này hướng dẫn cách truy cập
[metadata thiết bị](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#device-metadata)
từ các container sử dụng _cấp phát tài nguyên động (dynamic resource allocation — DRA)_.
Metadata thiết bị cho phép workload khám phá thông tin về các thiết bị đã được cấp phát,
chẳng hạn như các thuộc tính (attribute) của thiết bị hoặc chi tiết network interface — bằng
cách đọc các file JSON tại những đường dẫn quy ước (well-known path) bên trong container.

Trước khi đọc trang này, hãy làm quen với
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
và cách
[cấp phát thiết bị cho workload](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

* Hãy chắc chắn rằng quản trị viên cluster của bạn đã thiết lập DRA, gắn thiết bị và cài đặt
  driver. Để biết thêm thông tin, xem
  [Thiết lập DRA trong một cluster](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster).
* Hãy chắc chắn rằng DRA driver được triển khai trong cluster của bạn hỗ trợ metadata thiết bị.
  Các driver dùng [DRA kubelet plugin](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin)
  bật các tùy chọn `EnableDeviceMetadata` và
  `MetadataVersions` khi khởi động plugin. Hãy xem tài liệu của driver
  để biết chi tiết.

## Truy cập metadata thiết bị bằng ResourceClaim (Access device metadata with a ResourceClaim) {#access-metadata-resourceclaim}

Khi bạn dùng một ResourceClaim được tham chiếu trực tiếp để cấp phát thiết bị, các file
metadata thiết bị xuất hiện bên trong container tại:

```
/var/run/kubernetes.io/dra-device-attributes/resourceclaims/<claimName>/<requestName>/<driverName>-metadata.json
```

1. Xem lại manifest ví dụ sau:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: ResourceClaim
   metadata:
     name: gpu-claim
   spec:
     devices:
       requests:
       - name: gpu
         exactly:
           deviceClassName: gpu.example.com
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-metadata-reader
   spec:
     resourceClaims:
     - name: my-gpu
       resourceClaimName: gpu-claim
     containers:
     - name: workload
       image: ubuntu:24.04
       resources:
         claims:
         - name: my-gpu
           request: gpu
       command:
       - sh
       - -c
       - |
         echo "=== DRA device metadata ==="
         find /var/run/kubernetes.io/dra-device-attributes -name '*-metadata.json' -print -exec cat {} \;
         sleep 3600
     restartPolicy: Never
   ```

   Manifest này tạo một ResourceClaim tên là `gpu-claim` yêu cầu một
   thiết bị từ DeviceClass `gpu.example.com`, và một Pod đọc
   metadata của thiết bị.

1. Tạo ResourceClaim và Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-device-metadata-pod.yaml
   ```

1. Sau khi Pod chạy, xem log của container để thấy metadata:

   ```shell
   kubectl logs gpu-metadata-reader
   ```

   Output sẽ tương tự như sau:

   ```
   === DRA device metadata ===
   /var/run/kubernetes.io/dra-device-attributes/resourceclaims/gpu-claim/gpu/gpu.example.com-metadata.json
   {
     "kind": "DeviceMetadata",
     "apiVersion": "metadata.resource.k8s.io/v1alpha1",
     ...
   }
   ```

1. Để xem toàn bộ file metadata, hãy exec vào container:

   ```shell
   kubectl exec gpu-metadata-reader -- \
     cat /var/run/kubernetes.io/dra-device-attributes/resourceclaims/gpu-claim/gpu/gpu.example.com-metadata.json
   ```

   Output là một đối tượng JSON chứa các thuộc tính thiết bị như model,
   phiên bản driver và UUID của thiết bị. Xem
   [lược đồ metadata (metadata schema)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#device-metadata-schema)
   để biết chi tiết về cấu trúc JSON.

## Truy cập metadata thiết bị bằng ResourceClaimTemplate (Access device metadata with a ResourceClaimTemplate) {#access-metadata-template}

Khi bạn dùng một ResourceClaimTemplate, Kubernetes sinh ra một ResourceClaim cho
mỗi Pod. Vì tên của claim được sinh ra không thể đoán trước, các file metadata
xuất hiện tại một đường dẫn dùng tên tham chiếu claim trong Pod (Pod's claim
reference name) thay thế:

```
/var/run/kubernetes.io/dra-device-attributes/resourceclaimtemplates/<podClaimName>/<requestName>/<driverName>-metadata.json
```

`<podClaimName>` tương ứng với field `name` trong mục
`spec.resourceClaims[]` của Pod. JSON metadata cũng bao gồm một field
`podClaimName` ghi lại ánh xạ này.

1. Xem lại manifest ví dụ sau:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: ResourceClaimTemplate
   metadata:
     name: gpu-claim-template
   spec:
     spec:
       devices:
         requests:
         - name: gpu
           exactly:
             deviceClassName: gpu.example.com
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-metadata-template-reader
   spec:
     resourceClaims:
     - name: my-gpu
       resourceClaimTemplateName: gpu-claim-template
     containers:
     - name: workload
       image: ubuntu:24.04
       resources:
         claims:
         - name: my-gpu
           request: gpu
       command:
       - sh
       - -c
       - |
         echo "=== DRA device metadata (from template) ==="
         find /var/run/kubernetes.io/dra-device-attributes -name '*-metadata.json' -print -exec cat {} \;
         sleep 3600
     restartPolicy: Never
   ```

   Manifest này tạo một ResourceClaimTemplate và một Pod. Mỗi Pod nhận được
   một ResourceClaim riêng được sinh ra cho nó. Đường dẫn metadata dùng tên tham chiếu
   claim của Pod là `my-gpu`.

1. Tạo ResourceClaimTemplate và Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-device-metadata-template-pod.yaml
   ```

1. Sau khi Pod chạy, xem metadata:

   ```shell
   kubectl exec gpu-metadata-template-reader -- \
     cat /var/run/kubernetes.io/dra-device-attributes/resourceclaimtemplates/my-gpu/gpu/gpu.example.com-metadata.json
   ```

## Đọc metadata trong ứng dụng của bạn (Read metadata in your application) {#read-metadata-application}

### Ứng dụng Go (Go applications)

Package `k8s.io/dynamic-resource-allocation/devicemetadata` cung cấp các
hàm dựng sẵn để đọc các file metadata. Các hàm này tự động xử lý việc
thương lượng phiên bản (version negotiation), giải mã luồng metadata và chuyển đổi
nó sang các kiểu nội bộ, nhờ đó code của bạn hoạt động được trên nhiều phiên bản lược đồ
(schema version) mà không cần kiểm tra phiên bản thủ công.

Với một ResourceClaim được tham chiếu trực tiếp:

```go
import "k8s.io/dynamic-resource-allocation/devicemetadata"

dm, err := devicemetadata.ReadResourceClaimMetadata("gpu-claim", "gpu")
```

Với một claim được sinh từ template (dùng tên tham chiếu claim trong Pod):

```go
dm, err := devicemetadata.ReadResourceClaimTemplateMetadata("my-gpu", "gpu")
```

Nếu bạn biết tên driver cụ thể, bạn có thể đọc file metadata của một driver
duy nhất:

```go
dm, err := devicemetadata.ReadResourceClaimMetadataWithDriverName("gpu.example.com", "gpu-claim", "gpu")
```

Giá trị trả về `*metadata.DeviceMetadata` chứa metadata của claim, các request,
và các thuộc tính theo từng thiết bị.

Các ứng dụng viết bằng ngôn ngữ khác có thể đọc trực tiếp file JSON và kiểm tra
field `apiVersion` để xác định phiên bản lược đồ trước khi phân tích cú pháp (parse).

## Dọn dẹp (Clean up) {#clean-up}

Xóa các tài nguyên mà bạn đã tạo:

```shell
kubectl delete -f https://k8s.io/examples/dra/dra-device-metadata-pod.yaml
kubectl delete -f https://k8s.io/examples/dra/dra-device-metadata-template-pod.yaml
```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về metadata thiết bị DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#device-metadata)
* [Cấp phát thiết bị cho workload bằng DRA](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/)
* Để biết thêm thông tin về thiết kế, xem
  [KEP-5304](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5304-dra-attributes-downward-api).
