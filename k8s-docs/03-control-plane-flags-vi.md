# Tùy chỉnh các thành phần với kubeadm API (Customizing components with the kubeadm API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/>
>
> Trang này trình bày cách tùy chỉnh các thành phần mà kubeadm triển khai.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](LO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 3/9 ·
Kiểm chứng ở Lab 8a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Hai bài trước bạn dựng cluster bằng **cờ dòng lệnh** — đúng như
[A5.1 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) làm. Bài này là cách thứ
hai: mô tả cluster bằng **file cấu hình** rồi `kubeadm init --config`. Đọc nó như một bảng
phân vai — mỗi thứ muốn sửa có đúng một chỗ hợp lệ để sửa — chứ không phải để thuộc từng flag
trong các ví dụ YAML. Các flag cụ thể trong ví dụ (`audit-log-path`, `election-timeout`…) chỉ
là minh họa cú pháp.

**Phải hiểu ở lần đọc này:**

- *Tùy chỉnh control plane với các flag trong `ClusterConfiguration`*: bốn cấu trúc `apiServer`,
  `controllerManager`, `scheduler`, `etcd` cùng chia một trường `extraArgs`. `ClusterConfiguration`
  có **phạm vi toàn cục** — flag bạn thêm áp cho **mọi instance của thành phần đó trên mọi node**.
- `extraArgs` là danh sách cặp `name` / `value`, và **không hỗ trợ key trùng lặp** hay truyền
  cùng một flag nhiều lần. Nếu flag trỏ tới một file trên host thì phải mount file đó vào static
  Pod bằng `extraVolumes`, như ví dụ của scheduler.
- *Tùy chỉnh với các patch*: patch là **bước tùy chỉnh cuối cùng trước khi cấu hình của thành
  phần được ghi xuống đĩa**, khai báo bằng `patches.directory` và áp **trên từng node riêng lẻ**.
  Đây là lối thoát cho cả hai giới hạn ở trên: cấu hình khác nhau theo node, và flag trùng key.
- Quy ước tên file patch `target[suffix][+patchtype].extension`: `target` chỉ nhận
  `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`, `kubeletconfiguration`,
  `corednsdeployment`; `patchtype` mặc định là `strategic`; `extension` là `json` hoặc `yaml`.
- Ba thứ khác nhau đi qua ba kiểu cấu hình khác nhau: control plane qua `ClusterConfiguration`,
  kubelet qua `KubeletConfiguration` (hoặc `nodeRegistration.kubeletExtraArgs`), kube-proxy qua
  `KubeProxyConfiguration` — và vì **kubeadm triển khai kube-proxy dưới dạng DaemonSet** nên
  `KubeProxyConfiguration` luôn áp cho toàn bộ instance trong cluster.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung các flag ví dụ: `enable-admission-plugins`, `audit-log-path` | là cấu hình bảo mật và audit | giai đoạn 9, CP7 audit và mã hóa |
| `UpgradeConfiguration` với `apply:` / `node:` | chỉ dùng khi chạy `kubeadm upgrade` | CP2 nâng cấp |
| *Các flag của Etcd* (`election-timeout`) | tinh chỉnh etcd cần hiểu Raft và vận hành etcd | CP4 etcd backup |
| *Tùy chỉnh CoreDNS*, `dns.disabled: true` | thay CoreDNS là việc của vận hành DNS | CP6 DNS, CNI và kube-proxy |
| Ghi chú "Cấu hình lại một cluster kubeadm" ở đầu bài | sửa cluster **đang chạy** là quy trình khác | CP5 cấu hình lại cluster đang chạy |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Bạn muốn `kube-apiserver` trên control plane node thứ nhất ghi audit log vào
   `/var/log/a.log`, còn node thứ hai ghi vào `/var/log/b.log`. Đặt hai giá trị đó vào
   `ClusterConfiguration.apiServer.extraArgs` có ra kết quả mong muốn không? Vì sao?
2. Bạn thêm `- name: "config"` với `value: "/etc/kubernetes/scheduler-config.yaml"` vào
   `scheduler.extraArgs`. File đó có thật trên host, nhưng scheduler vẫn không đọc được. Ví dụ
   trong bài còn có thứ gì mà bạn bỏ sót?
3. Bạn đặt file `kube-apiserver0+merge.yaml` vào thư mục patch. Nó được áp vào thời điểm nào
   trong quy trình của kubeadm, và điều gì xảy ra với tùy chỉnh đó khi bạn chạy `kubeadm upgrade`?
4. Bạn muốn kube-proxy trên `k8s-worker2` chạy cấu hình khác hai node còn lại. Dùng
   `KubeProxyConfiguration` được không?
5. Lab 00 dựng cluster bằng cờ dòng lệnh. Nếu viết lại thành file cấu hình, bạn sẽ đặt cgroup
   driver của kubelet vào `kind` nào, và đặt `enable-admission-plugins` của API server vào
   `kind` nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** `ClusterConfiguration` **hiện có phạm vi toàn cục trong các cluster kubeadm**:
   bất kỳ flag nào bạn thêm vào cũng áp dụng cho **tất cả các instance của cùng một thành phần
   trên các node khác nhau**. Muốn cấu hình riêng cho từng node thì bài chỉ đúng một lối:
   dùng **patch**. Bẫy ở đây là cái tên — `ClusterConfiguration` nghe như "cấu hình của cluster
   này", nhưng phạm vi toàn cục nghĩa là bạn **không** mô tả được khác biệt giữa các node bằng nó.
2. **`extraVolumes`.** Các thành phần control plane chạy dưới dạng static Pod, nên một đường
   dẫn trên host không tự nhiên nhìn thấy được từ bên trong. Ví dụ scheduler trong bài đi kèm
   một mục `extraVolumes` với `name`, `hostPath`, `mountPath`, `readOnly` và `pathType: "File"`
   — chính là phần mount file cấu hình vào Pod. Thiếu nó thì flag trỏ tới một đường dẫn không
   tồn tại trong Pod.
3. Nó được áp **như bước tùy chỉnh cuối cùng, ngay trước khi cấu hình của thành phần được ghi
   xuống đĩa** — nghĩa là patch **thắng** mọi thứ kubeadm sinh ra trước đó, kể cả `extraArgs`.
   Khi nâng cấp, bài cảnh báo rõ: nếu dùng `kubeadm upgrade apply` và `kubeadm upgrade node`,
   bạn **phải cung cấp lại chính các patch đó** qua `UpgradeConfiguration`, nếu không phần tùy
   chỉnh **không được bảo toàn sau khi nâng cấp**. Phần `suffix` trong tên file chỉ để quyết
   định thứ tự áp theo alpha-numeric khi có nhiều patch cùng target.
4. **Không, không bằng `KubeProxyConfiguration`.** Bài ghi chú: kubeadm triển khai kube-proxy
   **dưới dạng một DaemonSet**, nên `KubeProxyConfiguration` sẽ áp dụng cho **tất cả** các
   instance kube-proxy trong cluster. Lưu ý `kube-proxy` cũng **không** nằm trong danh sách
   `target` hợp lệ của patch, khác với `kubeletconfiguration` — nên đây là ranh giới thật của
   cơ chế này, không phải chuyện chọn cú pháp nào cho tiện.
5. cgroup driver của kubelet nằm ở **`KubeletConfiguration`** — bài nói bạn thêm nó bên cạnh
   `ClusterConfiguration` hoặc `InitConfiguration`, phân tách bằng `---` trong cùng một file, và
   kubeadm sẽ áp cùng một `KubeletConfiguration` cơ sở cho mọi node. `enable-admission-plugins`
   là flag của kube-apiserver nên nằm ở **`ClusterConfiguration.apiServer.extraArgs`**. Hai thứ
   này không thay thế nhau được: `nodeRegistration.kubeletExtraArgs` cũng chỉnh được kubelet
   nhưng bài nhắc một số flag kubelet đã deprecated, nên phải kiểm tra tài liệu tham chiếu trước.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
