# Cấu hình lại một cluster kubeadm (Reconfiguring a kubeadm cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/>
>
> kubeadm không hỗ trợ cách tự động để cấu hình lại các thành phần đã được triển khai trên các
> node do nó quản lý. Trang này chỉ ra trình tự đúng của các bước cần thực hiện để cấu hình
> lại một cluster kubeadm.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — CP5 Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#cp5--cấu-hình-lại-cluster-đang-chạy),
bài 1/6 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này ngắn nhưng là **bản đồ tổng** của cả CP5: nó cho biết cấu hình của từng thành phần
(control plane, kubelet, kube-proxy, CoreDNS) nằm ở đâu, sửa thế nào, và — quan trọng nhất —
thứ gì sẽ bị `kubeadm upgrade` ghi đè nếu bạn không chủ động giữ lại. Các bài sau của CP5 đi
sâu vào từng trường hợp cụ thể.

**Phải hiểu ở lần đọc này:**

- Nguyên tắc chung: kubeadm **không tự động** cấu hình lại; cấu hình phạm vi cluster nằm trong
  các ConfigMap ở namespace `kube-system` (`kubeadm-config`, `kubelet-config`, `kube-proxy`),
  sửa bằng `kubectl edit` — và sau khi lưu, các thành phần đang chạy trên node **không** tự
  cập nhật theo.
- Cảnh báo validate: cấu hình trong ConfigMap được lưu dưới dạng chuỗi YAML phi cấu trúc, nên
  **không có bước validate** khi cập nhật — lỗi chính tả và lỗi thụt lề YAML lọt qua mà không
  bị chặn.
- Quy trình cho control plane: sửa `ClusterConfiguration` trong ConfigMap → sinh lại
  certificate (`kubeadm init phase certs`) hoặc manifest
  (`kubeadm init phase control-plane` / `kubeadm init phase etcd local`) với config file khớp
  nội dung mới; kubelet tự khởi động lại static Pod khi file manifest thay đổi; backup
  `/etc/kubernetes/` trước và làm **từng node một**.
- Quy trình cho kubelet: `kubeadm upgrade node phase kubelet-config` tải ConfigMap
  `kubelet-config` về `/var/lib/kubelet/config.yaml`; cấu hình riêng của node đặt qua flag
  trong `/var/lib/kubelet/kubeadm-flags.env`; kết thúc bằng `systemctl restart kubelet` — cũng
  từng node một.
- kube-proxy và CoreDNS, kèm cạm bẫy persist: sửa ConfigMap `kube-proxy` rồi xóa Pod theo
  label để tạo Pod mới; sửa Deployment `coredns`/Service `kube-dns` rồi rollout restart;
  nhưng `kubeadm upgrade` có thể **ghi đè** các thay đổi (CoreDNS, `/var/lib/kubelet/config.yaml`,
  Node object) — cách giữ lại: file patch cho static Pod, `kubectl patch --patch-file` cho
  Node object, flag trong `kubeadm-flags.env` cho kubelet.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Gợi ý tự động hóa việc cấu hình lại bằng một operator tùy chỉnh (phần mở đầu) | chỉ là một câu gợi ý hướng mở rộng, không có quy trình cụ thể | nền tảng đã có ở bài [181](181-operator-vi.md); tự tìm hiểu sâu khi có nhu cầu thật |
| Chi tiết cơ chế patch của kubeadm (`--patches`) | bài này chỉ nhắc tới; cú pháp và quy trình đầy đủ nằm ở bài khác | bài [03](03-control-plane-flags-vi.md#patches), mục *Tùy chỉnh với các patch* |

---

kubeadm không hỗ trợ các cách tự động để cấu hình lại các thành phần đã được triển khai trên
các node do nó quản lý. Một cách để tự động hóa việc này là dùng một
[operator](181-operator-vi.md) tùy chỉnh.

Để thay đổi cấu hình của các thành phần, bạn phải sửa thủ công các đối tượng cluster liên quan
và các file trên đĩa.

Hướng dẫn này chỉ ra trình tự đúng của các bước cần thực hiện để cấu hình lại một cluster
kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

- Bạn cần một cluster đã được triển khai bằng kubeadm
- Có credential quản trị viên (`/etc/kubernetes/admin.conf`) và kết nối mạng tới một
  kube-apiserver đang chạy trong cluster, từ một host đã cài kubectl
- Có một trình soạn thảo văn bản được cài trên tất cả các host

## Cấu hình lại cluster (Reconfiguring the cluster)

kubeadm ghi một tập các tùy chọn cấu hình thành phần ở phạm vi cluster vào các ConfigMap và
các đối tượng khác. Các đối tượng này phải được sửa thủ công. Lệnh `kubectl edit` có thể được
dùng cho việc đó.

Lệnh `kubectl edit` sẽ mở một trình soạn thảo văn bản, nơi bạn có thể sửa và lưu đối tượng
trực tiếp.

Bạn có thể dùng các biến môi trường `KUBECONFIG` và `KUBE_EDITOR` để chỉ định vị trí file
kubeconfig mà kubectl sử dụng và trình soạn thảo văn bản ưa thích.

Ví dụ:

```
KUBECONFIG=/etc/kubernetes/admin.conf KUBE_EDITOR=nano kubectl edit <parameters>
```

> **Ghi chú:** Sau khi lưu bất kỳ thay đổi nào vào các đối tượng cluster này, các thành phần
> đang chạy trên node có thể không được tự động cập nhật. Các bước bên dưới hướng dẫn bạn cách
> thực hiện việc đó thủ công.

> **Cảnh báo:** Cấu hình thành phần trong ConfigMap được lưu dưới dạng dữ liệu phi cấu trúc
> (chuỗi YAML). Điều này có nghĩa là việc validate sẽ không được thực hiện khi cập nhật nội
> dung của một ConfigMap. Bạn phải cẩn thận tuân theo định dạng API đã được tài liệu hóa cho
> cấu hình của từng thành phần cụ thể, và tránh đưa vào lỗi chính tả cũng như lỗi thụt lề YAML.

### Áp dụng các thay đổi cấu hình cluster (Applying cluster configuration changes)

#### Cập nhật `ClusterConfiguration` (Updating the `ClusterConfiguration`)

Trong quá trình tạo và nâng cấp cluster, kubeadm ghi
[`ClusterConfiguration`](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
của nó vào một ConfigMap tên là `kubeadm-config` trong namespace `kube-system`.

Để thay đổi một tùy chọn cụ thể trong `ClusterConfiguration`, bạn có thể sửa ConfigMap bằng
lệnh này:

```shell
kubectl edit cm -n kube-system kubeadm-config
```

Cấu hình nằm dưới key `data.ClusterConfiguration`.

> **Ghi chú:** `ClusterConfiguration` bao gồm nhiều tùy chọn ảnh hưởng tới cấu hình của các
> thành phần riêng lẻ như kube-apiserver, kube-scheduler, kube-controller-manager, CoreDNS,
> etcd và kube-proxy. Các thay đổi đối với cấu hình phải được phản ánh lên các thành phần trên
> node một cách thủ công.

#### Phản ánh các thay đổi `ClusterConfiguration` lên các node control plane (Reflecting `ClusterConfiguration` changes on control plane nodes)

kubeadm quản lý các thành phần control plane dưới dạng các manifest static Pod nằm trong thư
mục `/etc/kubernetes/manifests`.
Mọi thay đổi đối với `ClusterConfiguration` dưới các key `apiServer`, `controllerManager`,
`scheduler` hoặc `etcd` phải được phản ánh vào các file tương ứng trong thư mục manifests trên
một node control plane.

Những thay đổi như vậy có thể bao gồm:

- `extraArgs` - yêu cầu cập nhật danh sách flag được truyền cho container của thành phần
- `extraVolumes` - yêu cầu cập nhật các volume mount cho container của thành phần
- `*SANs` - yêu cầu ghi certificate mới với Subject Alternative Name được cập nhật

Trước khi tiến hành các thay đổi này, hãy đảm bảo bạn đã sao lưu (backup) thư mục
`/etc/kubernetes/`.

Để ghi certificate mới, bạn có thể dùng:

```shell
kubeadm init phase certs <component-name> --config <config-file>
```

Để ghi các file manifest mới trong `/etc/kubernetes/manifests`, bạn có thể dùng:

```shell
# Cho các thành phần control plane của Kubernetes
kubeadm init phase control-plane <component-name> --config <config-file>
# Cho etcd cục bộ
kubeadm init phase etcd local --config <config-file>
```

Nội dung của `<config-file>` phải khớp với `ClusterConfiguration` đã được cập nhật.
Giá trị `<component-name>` phải là tên của một thành phần control plane của Kubernetes
(`apiserver`, `controller-manager` hoặc `scheduler`).

> **Ghi chú:** Việc cập nhật một file trong `/etc/kubernetes/manifests` sẽ báo cho kubelet
> khởi động lại static Pod của thành phần tương ứng. Hãy cố gắng thực hiện các thay đổi này
> từng node một để cluster không bị gián đoạn (downtime).

### Áp dụng các thay đổi cấu hình kubelet (Applying kubelet configuration changes)

#### Cập nhật `KubeletConfiguration` (Updating the `KubeletConfiguration`)

Trong quá trình tạo và nâng cấp cluster, kubeadm ghi
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
của nó vào một ConfigMap tên là `kubelet-config` trong namespace `kube-system`.

Bạn có thể sửa ConfigMap bằng lệnh này:

```shell
kubectl edit cm -n kube-system kubelet-config
```

Cấu hình nằm dưới key `data.kubelet`.

#### Phản ánh các thay đổi của kubelet (Reflecting the kubelet changes)

Để phản ánh thay đổi lên các node kubeadm, bạn phải làm như sau:

- Đăng nhập vào một node kubeadm
- Chạy `kubeadm upgrade node phase kubelet-config` để tải nội dung mới nhất của ConfigMap
  `kubelet-config` về file cục bộ `/var/lib/kubelet/config.yaml`
- Sửa file `/var/lib/kubelet/kubeadm-flags.env` để áp dụng cấu hình bổ sung bằng các flag
- Khởi động lại dịch vụ kubelet với `systemctl restart kubelet`

> **Ghi chú:** Thực hiện các thay đổi này từng node một để các workload được lập lịch lại
> (reschedule) một cách hợp lý.

> **Ghi chú:** Trong quá trình `kubeadm upgrade`, kubeadm tải `KubeletConfiguration` từ
> ConfigMap `kubelet-config` xuống và ghi đè nội dung của `/var/lib/kubelet/config.yaml`.
> Điều này có nghĩa là cấu hình cục bộ của node phải được áp dụng hoặc bằng các flag trong
> `/var/lib/kubelet/kubeadm-flags.env`, hoặc bằng cách cập nhật thủ công nội dung của
> `/var/lib/kubelet/config.yaml` sau `kubeadm upgrade`, rồi khởi động lại kubelet.

### Áp dụng các thay đổi cấu hình kube-proxy (Applying kube-proxy configuration changes)

#### Cập nhật `KubeProxyConfiguration` (Updating the `KubeProxyConfiguration`)

Trong quá trình tạo và nâng cấp cluster, kubeadm ghi
[`KubeProxyConfiguration`](https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/)
của nó vào một ConfigMap trong namespace `kube-system` tên là `kube-proxy`.

ConfigMap này được sử dụng bởi DaemonSet `kube-proxy` trong namespace `kube-system`.

Để thay đổi một tùy chọn cụ thể trong `KubeProxyConfiguration`, bạn có thể sửa ConfigMap bằng
lệnh này:

```shell
kubectl edit cm -n kube-system kube-proxy
```

Cấu hình nằm dưới key `data.config.conf`.

#### Phản ánh các thay đổi của kube-proxy (Reflecting the kube-proxy changes)

Khi ConfigMap `kube-proxy` đã được cập nhật, bạn có thể khởi động lại tất cả các Pod
kube-proxy:

Xóa các Pod bằng:

```shell
kubectl delete po -n kube-system -l k8s-app=kube-proxy
```

Các Pod mới sử dụng ConfigMap đã được cập nhật sẽ được tạo ra.

> **Ghi chú:** Vì kubeadm triển khai kube-proxy dưới dạng một DaemonSet, cấu hình riêng theo
> từng node không được hỗ trợ.

### Áp dụng các thay đổi cấu hình CoreDNS (Applying CoreDNS configuration changes)

#### Cập nhật Deployment và Service của CoreDNS (Updating the CoreDNS Deployment and Service)

kubeadm triển khai CoreDNS dưới dạng một Deployment tên là `coredns` và với một Service
`kube-dns`, cả hai đều trong namespace `kube-system`.

Để cập nhật bất kỳ thiết lập nào của CoreDNS, bạn có thể sửa các đối tượng Deployment và
Service:

```shell
kubectl edit deployment -n kube-system coredns
kubectl edit service -n kube-system kube-dns
```

#### Phản ánh các thay đổi của CoreDNS (Reflecting the CoreDNS changes)

Khi các thay đổi CoreDNS đã được áp dụng, bạn có thể khởi động lại deployment CoreDNS:

```shell
kubectl rollout restart deployment -n kube-system coredns
```

> **Ghi chú:** kubeadm không cho phép cấu hình CoreDNS trong quá trình tạo và nâng cấp
> cluster. Điều này có nghĩa là nếu bạn thực thi `kubeadm upgrade apply`, các thay đổi của bạn
> đối với các đối tượng CoreDNS sẽ bị mất và phải được áp dụng lại.

## Duy trì cấu hình lại lâu dài (Persisting the reconfiguration)

Trong quá trình thực thi `kubeadm upgrade` trên một node do kubeadm quản lý, kubeadm có thể
ghi đè cấu hình đã được áp dụng sau khi cluster được tạo (việc cấu hình lại).

### Duy trì cấu hình lại của Node object (Persisting Node object reconfiguration)

kubeadm ghi các Label, Taint, CRI socket và các thông tin khác vào Node object của một node
Kubernetes cụ thể. Để thay đổi bất kỳ nội dung nào của Node object này, bạn có thể dùng:

```shell
kubectl edit no <node-name>
```

Trong quá trình `kubeadm upgrade`, nội dung của một Node như vậy có thể bị ghi đè.
Nếu bạn muốn giữ lại các sửa đổi của mình đối với Node object sau khi nâng cấp, bạn có thể
chuẩn bị một [kubectl patch](324-kubectl-patch-vi.md)
và áp dụng nó vào Node object:

```shell
kubectl patch no <node-name> --patch-file <patch-file>
```

#### Duy trì cấu hình lại của các thành phần control plane (Persisting control plane component reconfiguration)

Nguồn chính của cấu hình control plane là đối tượng `ClusterConfiguration` được lưu trong
cluster. Để mở rộng cấu hình của các manifest static Pod, có thể dùng
[các patch](03-control-plane-flags-vi.md#patches).

Các file patch này phải được giữ nguyên dưới dạng file trên các node control plane, để đảm bảo
chúng có thể được sử dụng bởi `kubeadm upgrade ... --patches <directory>`.

Nếu việc cấu hình lại được thực hiện trên `ClusterConfiguration` và các manifest static Pod
trên đĩa, thì tập các patch riêng của node phải được cập nhật tương ứng.

#### Duy trì cấu hình lại của kubelet (Persisting kubelet reconfiguration)

Mọi thay đổi đối với `KubeletConfiguration` được lưu trong `/var/lib/kubelet/config.yaml` sẽ
bị ghi đè khi chạy `kubeadm upgrade`, do nó tải xuống nội dung của ConfigMap `kubelet-config`
ở phạm vi cluster. Để duy trì cấu hình riêng của node cho kubelet, hoặc file
`/var/lib/kubelet/config.yaml` phải được cập nhật thủ công sau nâng cấp, hoặc file
`/var/lib/kubelet/kubeadm-flags.env` có thể chứa các flag.
Các flag của kubelet ghi đè các tùy chọn `KubeletConfiguration` tương ứng, nhưng lưu ý rằng
một số flag đã bị loại bỏ dần (deprecated).

Cần khởi động lại kubelet sau khi thay đổi `/var/lib/kubelet/config.yaml` hoặc
`/var/lib/kubelet/kubeadm-flags.env`.

## Tiếp theo (What's next)

- [Nâng cấp các cluster kubeadm](221-kubeadm-upgrade-vi.md)
- [Tùy chỉnh các thành phần với kubeadm API](03-control-plane-flags-vi.md)
- [Quản lý certificate với kubeadm](219-kubeadm-certs-vi.md)
- [Tìm hiểu thêm về thiết lập kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Bạn vừa sửa ConfigMap `kubelet-config` trong `kube-system` để đổi một tham số của kubelet,
   rồi ngồi chờ. Kubelet trên hai worker của bạn có tự nhận cấu hình mới không? Nếu không,
   chuỗi bước đúng trên mỗi node là gì, và vì sao phải làm từng node một?
2. `kubectl edit cm -n kube-system kubeadm-config` có kiểm tra tính hợp lệ (validate) của đoạn
   YAML `ClusterConfiguration` mà bạn nhập không? Điều đó dẫn tới nghĩa vụ gì cho bạn?
3. Bạn thêm một mục `extraArgs` cho kube-apiserver trong `ClusterConfiguration`. Kể các bước
   để thay đổi có hiệu lực trên node control plane, và giải thích vì sao không cần một lệnh
   "restart" riêng cho static Pod.
4. Trên `k8s-worker2`, bạn từng chỉnh tay `/var/lib/kubelet/config.yaml`. Sau lần
   `kubeadm upgrade` kế tiếp, chuyện gì xảy ra với chỉnh sửa đó, và bài đưa ra những cách nào
   để giữ lại cấu hình riêng của node?
5. Bạn đã chỉnh Deployment `coredns` và muốn kube-proxy trên riêng một node dùng cấu hình
   khác các node còn lại. Sau `kubeadm upgrade apply`, thay đổi CoreDNS còn không? Và yêu cầu
   về kube-proxy có khả thi không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không tự nhận.** Bài ghi rõ: sau khi lưu thay đổi vào các đối tượng cluster, các thành
   phần đang chạy trên node có thể không được tự động cập nhật. Chuỗi bước trên mỗi node: đăng
   nhập vào node → chạy `kubeadm upgrade node phase kubelet-config` để tải ConfigMap về
   `/var/lib/kubelet/config.yaml` → sửa `/var/lib/kubelet/kubeadm-flags.env` nếu cần cấu hình
   bổ sung bằng flag → `systemctl restart kubelet`. Làm **từng node một** để các workload được
   lập lịch lại một cách hợp lý.
2. **Không validate.** Cấu hình thành phần trong ConfigMap được lưu dưới dạng dữ liệu phi cấu
   trúc (chuỗi YAML), nên không có bước kiểm tra nào khi cập nhật. Nghĩa vụ của bạn: tự tuân
   theo đúng định dạng API đã tài liệu hóa của từng thành phần và tránh lỗi chính tả, lỗi thụt
   lề YAML — sai thì không ai chặn bạn lại.
3. Các bước: **backup `/etc/kubernetes/` trước** → sửa ConfigMap `kubeadm-config`
   (`kubectl edit cm -n kube-system kubeadm-config`, key `data.ClusterConfiguration`) → chạy
   `kubeadm init phase control-plane apiserver --config <config-file>` với nội dung config
   khớp `ClusterConfiguration` mới để ghi lại manifest trong `/etc/kubernetes/manifests`.
   Không cần lệnh restart riêng vì **việc cập nhật file trong `/etc/kubernetes/manifests` sẽ
   báo cho kubelet khởi động lại static Pod của thành phần tương ứng**. Làm từng node một để
   tránh downtime.
4. Chỉnh sửa đó **bị ghi đè**: trong `kubeadm upgrade`, kubeadm tải `KubeletConfiguration` từ
   ConfigMap `kubelet-config` và ghi đè `/var/lib/kubelet/config.yaml`. Hai cách giữ cấu hình
   riêng của node: (a) đặt nó dưới dạng **flag trong `/var/lib/kubelet/kubeadm-flags.env`**
   (flag ghi đè tùy chọn `KubeletConfiguration` tương ứng, nhưng một số flag đã deprecated),
   hoặc (b) **cập nhật thủ công lại `/var/lib/kubelet/config.yaml` sau nâng cấp**. Cả hai
   trường hợp đều phải khởi động lại kubelet sau khi thay đổi.
5. Thay đổi CoreDNS **mất**: kubeadm không cho phép cấu hình CoreDNS trong quá trình tạo và
   nâng cấp cluster, nên sau `kubeadm upgrade apply`, các thay đổi của bạn trên các đối tượng
   CoreDNS sẽ bị mất và phải áp dụng lại. Yêu cầu về kube-proxy **không khả thi**: kubeadm
   triển khai kube-proxy dưới dạng DaemonSet dùng chung một ConfigMap, nên cấu hình riêng theo
   từng node không được hỗ trợ. Trực giác "sửa trực tiếp Pod kube-proxy trên node đó là xong"
   sai vì DaemonSet sẽ tạo lại Pod từ template chung.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
