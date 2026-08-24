# Thiết lập DRA trong một cluster (Set Up DRA in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/
>
> Trang này hướng dẫn cách cấu hình cấp phát tài nguyên động (dynamic resource allocation — DRA)
> trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
> (classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này hướng dẫn cách cấu hình _cấp phát tài nguyên động (dynamic resource allocation — DRA)_
trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
(classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.

## Về DRA (About DRA) {#about-dra}

DRA là một tính năng của Kubernetes cho phép bạn yêu cầu (request) và chia sẻ tài nguyên
giữa các Pod. Các tài nguyên này thường là các thiết bị (device) gắn kèm,
chẳng hạn như bộ tăng tốc phần cứng (hardware accelerator).

Với DRA, các driver thiết bị và quản trị viên cluster định nghĩa các _lớp_ thiết bị
(device class) sẵn sàng để _claim_ (yêu cầu sử dụng) trong các workload. Kubernetes
cấp phát các thiết bị khớp với những claim cụ thể và đặt các Pod tương ứng
lên các node có thể truy cập những thiết bị đã được cấp phát.

Hãy chắc chắn rằng bạn đã quen với cách DRA hoạt động và với các thuật ngữ của DRA như
DeviceClass, ResourceClaim và ResourceClaimTemplate.
Để biết chi tiết, xem
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](149-dynamic-resource-allocation-vi.md).

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

* Gắn thiết bị vào cluster của bạn, trực tiếp hoặc gián tiếp. Để tránh các sự cố tiềm ẩn
  với driver, hãy đợi đến khi bạn thiết lập xong tính năng DRA cho cluster
  rồi mới cài đặt driver.

## Tùy chọn: bật thêm các API group của DRA (Optional: enable additional DRA API groups) {#enable-dra}

DRA nhìn chung là một tính năng ổn định (stable) trong Kubernetes; tuy nhiên, một số khía cạnh
của nó có thể vẫn đang ở giai đoạn alpha hoặc beta.
Nếu bạn muốn dùng bất kỳ khía cạnh nào của DRA chưa ổn định,
và tính năng liên quan dựa trên một kind API riêng,
thì bạn phải bật các API group alpha hoặc beta tương ứng.

Một số DRA driver hoặc workload cũ hơn có thể vẫn cần
API v1beta1 từ Kubernetes 1.30 hoặc v1beta2 từ Kubernetes 1.32.
Chỉ khi và chỉ nếu bạn muốn hỗ trợ chúng, hãy bật các
[API group](https://kubernetes.io/docs/reference/using-api/#api-groups) sau:

* `resource.k8s.io/v1beta1`
* `resource.k8s.io/v1beta2`

Các tính năng alpha có kiểu API riêng cần:

* `resource.k8s.io/v1alpha3`

Để biết thêm thông tin, xem
[Bật hoặc tắt các API group](https://kubernetes.io/docs/reference/using-api/#enabling-or-disabling).

## Xác minh rằng DRA đã được bật (Verify that DRA is enabled) {#verify}

Để xác minh rằng cluster đã được cấu hình đúng, hãy thử liệt kê các DeviceClass:

```shell
kubectl get deviceclasses
```

Nếu cấu hình của các thành phần là chính xác, output sẽ tương tự như sau:

```
No resources found
```

Nếu DRA chưa được cấu hình đúng, output của lệnh trên sẽ tương tự như sau:

```
error: the server doesn't have a resource type "deviceclasses"
```

Ví dụ, điều này có thể xảy ra khi API group resource.k8s.io bị tắt.
Một phép kiểm tra tương tự cũng áp dụng được cho các kiểu cấp cao nhất (top-level type)
ở chất lượng alpha hoặc beta.

Hãy thử các bước khắc phục sự cố sau:

1. Cấu hình lại và khởi động lại thành phần `kube-apiserver`.

1. Nếu toàn bộ field `.spec.resourceClaims` bị loại bỏ khỏi các Pod, hoặc nếu
   các Pod được lập lịch (schedule) mà không xét đến các ResourceClaim, hãy xác minh
   rằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
   `DynamicResourceAllocation` không bị tắt
   cho kube-apiserver, kube-controller-manager, kube-scheduler hoặc kubelet.

## Cài đặt driver thiết bị (Install device drivers) {#install-drivers}

Sau khi bạn bật DRA cho cluster, bạn có thể cài đặt các driver cho những thiết bị
đã gắn kèm. Để có hướng dẫn, hãy xem tài liệu của chủ sở hữu thiết bị
hoặc của dự án duy trì các driver thiết bị đó. Các driver mà bạn
cài đặt phải tương thích với DRA.

Để xác minh rằng các driver đã cài đặt đang hoạt động như mong đợi, hãy liệt kê
các ResourceSlice trong cluster của bạn:

```shell
kubectl get resourceslices
```

Output sẽ tương tự như sau:

```
NAME                                                  NODE                DRIVER               POOL                             AGE
00000-driver.example.com-cluster-1-node-1-abcde      cluster-1-node-1    driver.example.com   cluster-1-device-pool-1-r1gc     7s
00000-driver.example.com-cluster-1-node-2-fghij      cluster-1-node-2    driver.example.com   cluster-1-device-pool-2-446z     8s
```

Hãy thử các bước khắc phục sự cố sau:

1. Kiểm tra tình trạng (health) của DRA driver và tìm các thông báo lỗi về việc
   phát hành (publish) ResourceSlice trong log của nó. Nhà cung cấp driver
   có thể có thêm hướng dẫn về cài đặt và khắc phục sự cố.

## Tạo DeviceClass (Create DeviceClasses) {#create-deviceclasses}

Bạn có thể định nghĩa các nhóm thiết bị mà người vận hành ứng dụng có thể
claim trong workload bằng cách tạo các DeviceClass. Một số nhà cung cấp
driver thiết bị cũng có thể hướng dẫn bạn tạo DeviceClass trong quá trình
cài đặt driver.

Các ResourceSlice mà driver của bạn phát hành chứa thông tin về các
thiết bị mà driver đó quản lý, chẳng hạn như dung lượng (capacity), metadata và các thuộc tính
(attribute). Bạn có thể dùng CEL (Common Expression Language) để lọc theo các thuộc tính trong
DeviceClass của mình, điều này giúp người vận hành workload tìm thiết bị
dễ dàng hơn.

1.  Để tìm các thuộc tính thiết bị mà bạn có thể chọn trong DeviceClass bằng
    biểu thức CEL, hãy lấy đặc tả (specification) của một ResourceSlice:

    ```shell
    kubectl get resourceslice <resourceslice-name> -o yaml
    ```

    Output sẽ tương tự như sau:

    ```yaml
    apiVersion: resource.k8s.io/v1
    kind: ResourceSlice
    # các dòng được lược bỏ cho rõ ràng
    spec:
      devices:
      - attributes:
          type:
            string: gpu
        capacity:
          memory:
            value: 64Gi
        name: gpu-0
      - attributes:
          type:
            string: gpu
        capacity:
          memory:
            value: 64Gi
        name: gpu-1
      driver: driver.example.com
      nodeName: cluster-1-node-1
    # các dòng được lược bỏ cho rõ ràng
    ```

    Bạn cũng có thể xem tài liệu của nhà cung cấp driver để biết các thuộc tính
    và giá trị khả dụng.

1.  Xem lại manifest DeviceClass ví dụ sau, manifest này chọn mọi thiết bị
    được quản lý bởi driver thiết bị `driver.example.com`:

    ```yaml
    apiVersion: resource.k8s.io/v1
    kind: DeviceClass
    metadata:
      name: example-device-class
    spec:
      selectors:
      - cel:
          expression: |-
            device.driver == "driver.example.com"
    ```

1.  Tạo DeviceClass trong cluster của bạn:

    ```shell
    kubectl apply -f https://k8s.io/examples/dra/deviceclass.yaml
    ```

## Dọn dẹp (Clean up) {#clean-up}

Để xóa DeviceClass mà bạn đã tạo trong bài thực hành này, chạy lệnh sau:

```shell
kubectl delete -f https://k8s.io/examples/dra/deviceclass.yaml
```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về DRA](149-dynamic-resource-allocation-vi.md)
* [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md)
