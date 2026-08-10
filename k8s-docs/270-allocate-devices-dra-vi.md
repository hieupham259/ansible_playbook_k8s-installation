# Cấp phát thiết bị cho workload bằng DRA (Allocate Devices to Workloads with DRA)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/
>
> Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng cấp phát tài nguyên động
> (dynamic resource allocation — DRA). Các hướng dẫn này dành cho người vận hành workload.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng
_cấp phát tài nguyên động (dynamic resource allocation — DRA)_. Các hướng dẫn này dành cho
người vận hành workload (workload operator). Trước khi đọc trang này, hãy làm quen với cách
DRA hoạt động và với các thuật ngữ của DRA như
ResourceClaim và ResourceClaimTemplate.
Để biết thêm thông tin, xem
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/).

## Về việc cấp phát thiết bị bằng DRA (About device allocation with DRA) {#about-device-allocation-dra}

Với vai trò người vận hành workload, bạn có thể _claim_ (yêu cầu sử dụng) thiết bị cho các
workload của mình bằng cách tạo ResourceClaim hoặc ResourceClaimTemplate. Khi bạn triển khai
workload, Kubernetes và các driver thiết bị sẽ tìm thiết bị khả dụng, cấp phát chúng cho các
Pod của bạn, và đặt các Pod lên những node có thể truy cập các thiết bị đó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.34 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

* Hãy chắc chắn rằng quản trị viên cluster của bạn đã thiết lập DRA, gắn thiết bị và cài đặt
  driver. Để biết thêm thông tin, xem
  [Thiết lập DRA trong một cluster](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster).

## Nhận diện thiết bị cần claim (Identify devices to claim) {#identify-devices}

Quản trị viên cluster của bạn hoặc các driver thiết bị sẽ tạo các
_DeviceClass_ định nghĩa các nhóm thiết bị. Bạn có thể claim thiết bị bằng cách dùng
CEL (Common Expression Language) để lọc theo các thuộc tính (property) cụ thể của thiết bị.

Lấy danh sách các DeviceClass trong cluster:

```shell
kubectl get deviceclasses
```

Output sẽ tương tự như sau:

```
NAME                 AGE
driver.example.com   16m
```

Nếu bạn gặp lỗi về quyền (permission error), có thể bạn không có quyền lấy danh sách DeviceClass.
Hãy hỏi quản trị viên cluster hoặc nhà cung cấp driver về các thuộc tính thiết bị
khả dụng.

## Claim tài nguyên (Claim resources) {#claim-resources}

Bạn có thể yêu cầu tài nguyên từ một DeviceClass bằng cách dùng
ResourceClaim. Để tạo một ResourceClaim, hãy làm một trong các cách sau:

* Tạo thủ công một ResourceClaim nếu bạn muốn nhiều Pod chia sẻ quyền truy cập
  cùng một nhóm thiết bị, hoặc nếu bạn muốn một claim tồn tại lâu hơn vòng đời của một
  Pod.
* Dùng một
  ResourceClaimTemplate
  để Kubernetes tự sinh và quản lý ResourceClaim riêng cho từng Pod. Hãy tạo
  ResourceClaimTemplate nếu bạn muốn mỗi Pod có quyền truy cập vào các thiết bị riêng biệt
  nhưng có cấu hình tương tự nhau. Ví dụ, bạn có thể muốn truy cập đồng thời
  vào thiết bị cho các Pod trong một Job dùng
  [thực thi song song (parallel execution)](https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs).

Nếu bạn tham chiếu trực tiếp một ResourceClaim cụ thể trong một Pod, ResourceClaim đó
phải đã tồn tại trong cluster. Nếu ResourceClaim được tham chiếu không tồn tại,
Pod sẽ ở trạng thái pending cho đến khi ResourceClaim được tạo. Bạn có thể
tham chiếu một ResourceClaim được tự động sinh ra trong một Pod, nhưng điều này không được
khuyến nghị vì các ResourceClaim tự động sinh bị ràng buộc vào vòng đời của Pod
đã kích hoạt việc sinh ra chúng.

Để tạo một workload claim tài nguyên, hãy chọn một trong các phương án sau:

#### ResourceClaimTemplate

Xem lại manifest ví dụ sau:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: example-resource-claim-template
spec:
  spec:
    devices:
      requests:
      - name: gpu-claim
        exactly:
          deviceClassName: example-device-class
          selectors:
            - cel:
                expression: |-
                  device.attributes["driver.example.com"].type == "gpu" &&
                  device.capacity["driver.example.com"].memory == quantity("64Gi")
```

Manifest này tạo một ResourceClaimTemplate yêu cầu các thiết bị trong
DeviceClass `example-device-class` khớp cả hai điều kiện sau:

* Thiết bị có thuộc tính `driver.example.com/type` với giá trị
  `gpu`.
* Thiết bị có dung lượng (capacity) `64Gi`.

Để tạo ResourceClaimTemplate, chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
```

#### ResourceClaim

Xem lại manifest ví dụ sau:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: example-resource-claim
spec:
  devices:
    requests:
    - name: single-gpu-claim
      exactly:
        deviceClassName: example-device-class
        allocationMode: All
        selectors:
        - cel:
            expression: |-
              device.attributes["driver.example.com"].type == "gpu" &&
              device.capacity["driver.example.com"].memory == quantity("64Gi")
```

Manifest này tạo một ResourceClaim yêu cầu các thiết bị trong
DeviceClass `example-device-class` khớp cả hai điều kiện sau:

* Thiết bị có thuộc tính `driver.example.com/type` với giá trị
  `gpu`.
* Thiết bị có dung lượng (capacity) `64Gi`.

Để tạo ResourceClaim, chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaim.yaml
```

## Yêu cầu thiết bị trong workload bằng DRA (Request devices in workloads using DRA) {#request-devices-workloads}

Để yêu cầu cấp phát thiết bị, hãy chỉ định một ResourceClaim hoặc một ResourceClaimTemplate
trong field `resourceClaims` của đặc tả Pod. Sau đó, yêu cầu một
claim cụ thể theo tên trong field `resources.claims` của một container trong Pod đó.
Bạn có thể chỉ định nhiều mục trong field `resourceClaims` và dùng các
claim cụ thể trong các container khác nhau.

1. Xem lại Job ví dụ sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: example-dra-job
   spec:
     completions: 10
     parallelism: 2
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: container0
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: separate-gpu-claim
         - name: container1
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: shared-gpu-claim
         - name: container2
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: shared-gpu-claim
         resourceClaims:
         - name: separate-gpu-claim
           resourceClaimTemplateName: example-resource-claim-template
         - name: shared-gpu-claim
           resourceClaimName: example-resource-claim
   ```

   Mỗi Pod trong Job này có các đặc điểm sau:

   * Cung cấp cho các container một ResourceClaimTemplate tên là `separate-gpu-claim` và một
     ResourceClaim tên là `shared-gpu-claim`.
   * Chạy các container sau:
       * `container0` yêu cầu các thiết bị từ ResourceClaimTemplate
         `separate-gpu-claim`.
       * `container1` và `container2` chia sẻ quyền truy cập vào các thiết bị từ
         ResourceClaim `shared-gpu-claim`.

1. Tạo Job:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-example-job.yaml
   ```

Hãy thử các bước khắc phục sự cố sau:

1. Khi workload không khởi động như mong đợi, hãy đi sâu dần từ Job
   xuống các Pod rồi tới các ResourceClaim, và kiểm tra các đối tượng
   ở từng cấp bằng `kubectl describe` để xem có field trạng thái hay
   sự kiện (event) nào giải thích vì sao workload
   không khởi động hay không.
1. Khi việc tạo Pod thất bại với lỗi `must specify one of: resourceClaimName,
   resourceClaimTemplateName`, hãy kiểm tra rằng mọi mục trong `pod.spec.resourceClaims`
   đều có đúng một trong hai field đó được đặt. Nếu chúng đã đúng, thì có khả năng
   cluster đã cài một mutating Pod webhook được build
   dựa trên các API của Kubernetes < 1.32. Hãy làm việc với quản trị viên cluster
   để kiểm tra điều này.

## Dọn dẹp (Clean up) {#clean-up}

Để xóa các đối tượng Kubernetes mà bạn đã tạo trong bài thực hành này, làm theo các
bước sau:

1.  Xóa Job ví dụ:

    ```shell
    kubectl delete -f https://k8s.io/examples/dra/dra-example-job.yaml
    ```

1.  Để xóa các resource claim của bạn, chạy một trong các lệnh sau:

    * Xóa ResourceClaimTemplate:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
      ```
    * Xóa ResourceClaim:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaim.yaml
      ```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation)
