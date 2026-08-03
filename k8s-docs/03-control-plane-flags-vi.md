# Tùy chỉnh các thành phần với kubeadm API (Customizing components with the kubeadm API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/>
>
> Trang này trình bày cách tùy chỉnh các thành phần mà kubeadm triển khai.

Trang này trình bày cách tùy chỉnh các thành phần mà kubeadm triển khai. Đối với các thành phần control plane,
bạn có thể sử dụng các flag trong cấu trúc `ClusterConfiguration` hoặc các patch cho từng node. Đối với kubelet
và kube-proxy, bạn có thể sử dụng `KubeletConfiguration` và `KubeProxyConfiguration` tương ứng.

Tất cả các tùy chọn này đều khả dụng thông qua API cấu hình của kubeadm.
Để biết thêm chi tiết về từng trường (field) trong cấu hình, bạn có thể truy cập
[các trang tham chiếu API](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của chúng tôi.

> **Ghi chú:**
> Để cấu hình lại một cluster đã được tạo từ trước, hãy xem
> [Cấu hình lại một cluster kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure).

## Tùy chỉnh control plane với các flag trong `ClusterConfiguration` (Customizing the control plane with flags in `ClusterConfiguration`)

Đối tượng `ClusterConfiguration` của kubeadm cung cấp cho người dùng một cách để ghi đè (override) các flag mặc định
được truyền cho các thành phần control plane như APIServer, ControllerManager, Scheduler và Etcd.
Các thành phần này được định nghĩa bằng các cấu trúc sau:

- `apiServer`
- `controllerManager`
- `scheduler`
- `etcd`

Các cấu trúc này có chung một trường `extraArgs`, bao gồm các cặp `name` / `value`.
Để ghi đè một flag cho một thành phần control plane:

1.  Thêm trường `extraArgs` phù hợp vào cấu hình của bạn.
2.  Thêm các flag vào trường `extraArgs`.
3.  Chạy `kubeadm init` với `--config <FILE YAML CẤU HÌNH CỦA BẠN>`.

> **Ghi chú:**
> Bạn có thể tạo một đối tượng `ClusterConfiguration` với các giá trị mặc định bằng cách chạy `kubeadm config print init-defaults`
> và lưu kết quả vào một file tùy chọn.

> **Ghi chú:**
> Đối tượng `ClusterConfiguration` hiện tại có phạm vi toàn cục (global) trong các cluster kubeadm. Điều này có nghĩa là bất kỳ flag nào bạn thêm vào
> sẽ áp dụng cho tất cả các instance của cùng một thành phần trên các node khác nhau. Để áp dụng cấu hình riêng cho từng thành phần
> trên các node khác nhau, bạn có thể sử dụng [patch](#patches).

> **Ghi chú:**
> Các flag (key) trùng lặp, hoặc truyền cùng một flag `--foo` nhiều lần, hiện chưa được hỗ trợ.
> Để giải quyết vấn đề đó, bạn phải sử dụng [patch](#patches).

### Các flag của APIServer (APIServer flags)

Để biết chi tiết, hãy xem [tài liệu tham chiếu cho kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

Ví dụ sử dụng:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
apiServer:
  extraArgs:
  - name: "enable-admission-plugins"
    value: "AlwaysPullImages,DefaultStorageClass"
  - name: "audit-log-path"
    value: "/home/johndoe/audit.log"
```

### Các flag của ControllerManager (ControllerManager flags)

Để biết chi tiết, hãy xem [tài liệu tham chiếu cho kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

Ví dụ sử dụng:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
controllerManager:
  extraArgs:
  - name: "cluster-signing-key-file"
    value: "/home/johndoe/keys/ca.key"
  - name: "deployment-controller-sync-period"
    value: "50"
```

### Các flag của Scheduler (Scheduler flags)

Để biết chi tiết, hãy xem [tài liệu tham chiếu cho kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/).

Ví dụ sử dụng:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
scheduler:
  extraArgs:
  - name: "config"
    value: "/etc/kubernetes/scheduler-config.yaml"
  extraVolumes:
    - name: schedulerconfig
      hostPath: /home/johndoe/schedconfig.yaml
      mountPath: /etc/kubernetes/scheduler-config.yaml
      readOnly: true
      pathType: "File"
```

### Các flag của Etcd (Etcd flags)

Để biết chi tiết, hãy xem [tài liệu về etcd server](https://etcd.io/docs/).

Ví dụ sử dụng:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
etcd:
  local:
    extraArgs:
    - name: "election-timeout"
      value: 1000
```

## Tùy chỉnh với các patch (Customizing with patches) {#patches}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [beta]`

Kubeadm cho phép bạn truyền một thư mục chứa các file patch vào `InitConfiguration`,
`JoinConfiguration` và `UpgradeConfiguration` trên từng node riêng lẻ. Các patch này có thể được sử dụng như bước tùy chỉnh cuối cùng
trước khi cấu hình của thành phần được ghi xuống đĩa.

Bạn có thể truyền file này cho `kubeadm init` với `--config <FILE YAML CẤU HÌNH CỦA BẠN>`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
patches:
  directory: /home/user/somedir
```

> **Ghi chú:**
> Đối với `kubeadm init`, bạn có thể truyền một file chứa cả `ClusterConfiguration` và `InitConfiguration`
> được phân tách bằng `---`.

Bạn có thể truyền file này cho `kubeadm join` với `--config <FILE YAML CẤU HÌNH CỦA BẠN>`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
patches:
  directory: /home/user/somedir
```

Nếu bạn đang sử dụng `kubeadm upgrade apply` và `kubeadm upgrade node` để nâng cấp các node kubeadm,
bạn phải cung cấp lại chính các patch đó, để phần tùy chỉnh được bảo toàn sau khi nâng cấp.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: UpgradeConfiguration
apply:
  patches:
    directory: /home/user/somedir
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: UpgradeConfiguration
node:
  patches:
    directory: /home/user/somedir
```

Thư mục này phải chứa các file được đặt tên theo dạng `target[suffix][+patchtype].extension`.
Ví dụ, `kube-apiserver0+merge.yaml` hoặc chỉ đơn giản là `etcd.json`.

- `target` có thể là một trong các giá trị `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`,
`kubeletconfiguration` và `corednsdeployment`.
- `suffix` là một chuỗi tùy chọn có thể được dùng để xác định patch nào được áp dụng trước
theo thứ tự chữ-số (alpha-numerically).
- `patchtype` có thể là một trong các giá trị `strategic`, `merge` hoặc `json` và chúng phải khớp với các định dạng patch
[được kubectl hỗ trợ](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch).
`patchtype` mặc định là `strategic`.
- `extension` phải là `json` hoặc `yaml`.

## Tùy chỉnh kubelet (Customizing the kubelet) {#kubelet}

Để tùy chỉnh kubelet, bạn có thể thêm một [`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
bên cạnh `ClusterConfiguration` hoặc `InitConfiguration`, phân tách bằng `---` trong cùng một file cấu hình.
File này sau đó có thể được truyền cho `kubeadm init` và kubeadm sẽ áp dụng cùng một `KubeletConfiguration` cơ sở
cho tất cả các node trong cluster.

Để áp dụng cấu hình riêng cho từng instance đè lên `KubeletConfiguration` cơ sở, bạn có thể sử dụng
[patch target `kubeletconfiguration`](#patches).

Ngoài ra, bạn có thể sử dụng các flag của kubelet để ghi đè bằng cách truyền chúng vào trường
`nodeRegistration.kubeletExtraArgs` được hỗ trợ bởi cả `InitConfiguration` và `JoinConfiguration`.
Một số flag của kubelet đã bị loại bỏ dần (deprecated), vì vậy hãy kiểm tra trạng thái của chúng trong
[tài liệu tham chiếu kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet) trước khi sử dụng.

Để biết thêm chi tiết, hãy xem [Cấu hình từng kubelet trong cluster của bạn bằng kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration)

## Tùy chỉnh kube-proxy (Customizing kube-proxy)

Để tùy chỉnh kube-proxy, bạn có thể truyền một `KubeProxyConfiguration` bên cạnh `ClusterConfiguration` hoặc
`InitConfiguration` cho `kubeadm init`, phân tách bằng `---`.

Để biết thêm chi tiết, bạn có thể truy cập [các trang tham chiếu API](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của chúng tôi.

> **Ghi chú:**
> kubeadm triển khai kube-proxy dưới dạng một DaemonSet, điều này có nghĩa là
> `KubeProxyConfiguration` sẽ áp dụng cho tất cả các instance của kube-proxy trong cluster.

## Tùy chỉnh CoreDNS (Customizing CoreDNS)

kubeadm cho phép bạn tùy chỉnh CoreDNS Deployment bằng các patch nhắm vào
[patch target `corednsdeployment`](#patches).

Các patch cho những đối tượng API khác liên quan đến CoreDNS như ConfigMap `kube-system/coredns`
hiện chưa được hỗ trợ.
Bạn phải tự patch các đối tượng này bằng kubectl và tạo lại các Pod CoreDNS sau đó.

Ngoài ra, bạn có thể vô hiệu hóa việc kubeadm triển khai CoreDNS bằng cách thêm tùy chọn sau
vào `ClusterConfiguration` của bạn:

```yaml
dns:
  disabled: true
```

Đồng thời, bằng cách thực thi lệnh sau:

```shell
kubeadm init phase addon coredns --print-manifest --config my-config.yaml`
```

bạn có thể lấy được file manifest mà kubeadm sẽ tạo cho CoreDNS trên hệ thống của bạn.
