# Lập lịch GPU (Schedule GPUs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/>
>
> Cấu hình và lập lịch GPU để dùng như một resource của các node trong cluster.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Kubernetes có hỗ trợ **ổn định (stable)** cho việc quản lý GPU (graphical processing unit —
bộ xử lý đồ họa) của AMD và NVIDIA trên các node khác nhau trong cluster của bạn, thông qua
device plugin.

Trang này mô tả cách người dùng sử dụng GPU, đồng thời nêu ra một số giới hạn trong cách
triển khai hiện tại.

## Sử dụng device plugin (Using device plugins)

Kubernetes triển khai device plugin để cho phép các Pod truy cập những tính năng phần cứng
chuyên dụng, chẳng hạn như GPU.

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn
> được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Với vai trò quản trị viên (administrator), bạn phải cài driver GPU của nhà cung cấp phần cứng
tương ứng lên các node và chạy device plugin tương ứng do nhà cung cấp GPU đó phát hành. Dưới
đây là một số liên kết tới hướng dẫn của các nhà cung cấp:

* [AMD](https://github.com/ROCm/k8s-device-plugin#deployment)
* [Intel](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html)
* [NVIDIA](https://github.com/NVIDIA/k8s-device-plugin#quick-start)

Sau khi bạn đã cài plugin, cluster của bạn sẽ expose một resource tùy chỉnh có thể lập lịch
được, chẳng hạn `amd.com/gpu` hoặc `nvidia.com/gpu`.

Bạn có thể sử dụng những GPU này từ trong container bằng cách yêu cầu resource GPU tùy chỉnh
đó, hoàn toàn giống cách bạn yêu cầu `cpu` hay `memory`. Tuy nhiên, có một vài giới hạn trong
cách bạn khai báo yêu cầu resource cho các thiết bị tùy chỉnh.

GPU chỉ được phép khai báo trong phần `limits`, điều đó có nghĩa là:
* Bạn có thể khai báo `limits` cho GPU mà không cần khai báo `requests`, bởi vì Kubernetes sẽ
  mặc định lấy giá trị limit làm giá trị request.
* Bạn có thể khai báo GPU ở cả `limits` lẫn `requests`, nhưng hai giá trị này bắt buộc phải
  bằng nhau.
* Bạn không thể khai báo `requests` cho GPU mà không khai báo `limits`.

Dưới đây là một manifest ví dụ cho Pod yêu cầu một GPU:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: example-vector-add
      image: "registry.example/example-vector-add:v42"
      resources:
        limits:
          gpu-vendor.example/example-gpu: 1 # yêu cầu 1 GPU
```

## Quản lý cluster có nhiều loại GPU khác nhau (Manage clusters with different types of GPUs)

Nếu các node khác nhau trong cluster của bạn có các loại GPU khác nhau, bạn có thể dùng
[Node Label và Node Selector](267-assign-pods-nodes-vi.md) để lập lịch pod lên đúng node phù hợp.

Ví dụ:

```shell
# Gắn label cho các node theo loại accelerator mà chúng có.
kubectl label nodes node1 accelerator=example-gpu-x100
kubectl label nodes node2 accelerator=other-gpu-k915
```

Khóa label `accelerator` ở trên chỉ là một ví dụ; bạn có thể dùng một khóa label khác nếu muốn.

## Tự động gắn label cho node (Automatic node labelling) {#node-labeller}

Với vai trò quản trị viên, bạn có thể tự động phát hiện và gắn label cho tất cả các node có GPU
bằng cách triển khai [Node Feature Discovery](https://github.com/kubernetes-sigs/node-feature-discovery)
(NFD) của Kubernetes. NFD phát hiện các tính năng phần cứng sẵn có trên từng node trong một
cluster Kubernetes. Thông thường, NFD được cấu hình để quảng bá những tính năng đó dưới dạng
node label, nhưng NFD cũng có thể thêm extended resource, annotation và node taint. NFD tương
thích với mọi [phiên bản được hỗ trợ](https://kubernetes.io/releases/version-skew-policy/#supported-versions)
của Kubernetes. Theo mặc định, NFD tạo các
[feature label](https://kubernetes-sigs.github.io/node-feature-discovery/master/usage/features.html)
cho những tính năng mà nó phát hiện được. Quản trị viên có thể tận dụng NFD để đồng thời gắn
taint cho các node có tính năng cụ thể, sao cho chỉ những pod yêu cầu các tính năng đó mới được
lập lịch lên những node ấy.

Bạn cũng cần một plugin cho NFD để thêm các label phù hợp vào node của mình; đó có thể là các
label chung chung hoặc các label riêng của từng nhà cung cấp. Nhà cung cấp GPU của bạn có thể
cung cấp một plugin bên thứ ba cho NFD; hãy xem tài liệu của họ để biết thêm chi tiết.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-vector-add
spec:
  restartPolicy: OnFailure
  # Bạn có thể dùng node affinity của Kubernetes để lập lịch Pod này lên một node
  # cung cấp đúng loại GPU mà container của nó cần để hoạt động
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "gpu.gpu-vendor.example/installed-memory"
            operator: Gt # (lớn hơn)
            values: ["40535"]
          - key: "feature.node.kubernetes.io/pci-10.present" # Feature label của NFD
            values: ["true"] # (tùy chọn) chỉ lập lịch lên các node có thiết bị PCI 10
  containers:
    - name: example-vector-add
      image: "registry.example/example-vector-add:v42"
      resources:
        limits:
          gpu-vendor.example/example-gpu: 1 # yêu cầu 1 GPU
```

#### Các triển khai của nhà cung cấp GPU (GPU vendor implementations)

- [Intel](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html)
- [NVIDIA](https://github.com/NVIDIA/k8s-device-plugin)
