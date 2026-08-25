# Bật hoặc tắt Feature Gate (Enable Or Disable Feature Gates)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/configure-feature-gates/>
>
> Trang này hướng dẫn cách bật hoặc tắt các feature gate để điều khiển những tính năng cụ thể
> của Kubernetes trong cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — giai đoạn 20 Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
bài 4/6 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này là mặt "thao tác" của khái niệm feature gate mà bạn đã gặp rải rác suốt lộ trình mỗi
khi thấy dòng **TRẠNG THÁI TÍNH NĂNG**. Điểm cần nắm không phải là một gate cụ thể nào, mà là
**bản đồ các vị trí cấu hình** trong một cluster kubeadm — thứ bạn sẽ dùng lại mỗi lần muốn
thử một tính năng Alpha/Beta.

**Phải hiểu ở lần đọc này:**

- Ba mức trưởng thành và mặc định của chúng: **Alpha tắt mặc định** (chỉ dùng ở cluster thử
  nghiệm), **Beta thường bật mặc định**, **GA luôn bật mặc định** — GA đôi khi tắt được trong
  một minor release sau khi lên GA, nhưng làm vậy cluster có thể không còn "conformant".
- Feature gate là cấu hình **theo từng thành phần**: có tính năng cần bật gate ở nhiều thành
  phần, có tính năng chỉ cần một; mọi thành phần dùng chung định nghĩa gate nên gate nào cũng
  hiện trong help, nhưng chỉ gate liên quan mới ảnh hưởng hành vi thành phần đó.
- Cách bật lúc khởi tạo cluster: một file cấu hình kubeadm gồm `ClusterConfiguration`
  (`extraArgs` cho apiServer/controllerManager/scheduler) và `KubeletConfiguration`
  (`featureGates`), đưa vào `kubeadm init --config`.
- Cách bật trên cluster đang chạy, ba đường khác nhau: sửa static pod manifest trong
  `/etc/kubernetes/manifests/` (pod tự restart), sửa `/var/lib/kubelet/config.yaml` rồi restart
  kubelet, sửa ConfigMap `kube-proxy` rồi rollout restart DaemonSet.
- Cách kiểm chứng: đọc manifest static pod, endpoint configz của kubelet, và metric
  `kubernetes_feature_enabled` (giá trị `1`/`0`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Kiểm tra qua endpoint /flagz* (z-pages) | cần bật gate `ComponentFlagz` và có quyền truy cập endpoint gỡ lỗi của thành phần | tài liệu [z-pages](https://kubernetes.io/docs/reference/instrumentation/zpages/); nền tảng về quan sát thành phần ở bài [160](160-system-metrics-vi.md) |

---

Trang này hướng dẫn cách bật hoặc tắt các feature gate để điều khiển những tính năng cụ thể
của Kubernetes trong cluster của bạn. Việc bật feature gate cho phép bạn thử nghiệm và sử dụng
các tính năng Alpha hoặc Beta trước khi chúng trở nên phổ biến rộng rãi (generally available).

> **Ghi chú:** Với một số gate đã ổn định (GA), bạn cũng có thể tắt chúng, thường là trong một
> minor release sau khi lên GA; tuy nhiên nếu làm vậy, cluster của bạn có thể không còn đạt
> chuẩn tương thích (conformant) của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò host của control plane. Nếu chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cũng cần:

* Quyền quản trị (administrative access) đối với cluster của bạn
* Biết mình muốn bật feature gate nào (xem [tài liệu tham khảo Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/))

> **Ghi chú:** Các tính năng GA (ổn định) luôn được bật theo mặc định. Bạn thường chỉ cấu hình
> gate cho các tính năng Alpha hoặc Beta.

## Hiểu mức độ trưởng thành của feature gate (Understand feature gate maturity)

Trước khi bật một feature gate, hãy kiểm tra
[tài liệu tham khảo Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
để biết mức độ trưởng thành của tính năng:

- **Alpha**: Tắt theo mặc định, có thể còn lỗi. Chỉ dùng trong các cluster thử nghiệm.
- **Beta**: Thường bật theo mặc định, đã được kiểm thử kỹ.
- **GA**: Luôn bật theo mặc định; đôi khi có thể tắt trong một release sau khi lên GA.

## Xác định thành phần nào cần feature gate (Identify which components need the feature gate)

Các feature gate khác nhau ảnh hưởng đến các thành phần Kubernetes khác nhau:

- Một số tính năng đòi hỏi bật gate trên **nhiều thành phần** (ví dụ: API server và
  controller manager)
- Các tính năng khác chỉ cần gate trên **một thành phần duy nhất** (ví dụ: chỉ kubelet)

[Tài liệu tham khảo Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
thường chỉ ra thành phần nào bị ảnh hưởng bởi từng gate. Tất cả các thành phần Kubernetes dùng
chung cùng một bộ định nghĩa feature gate, vì thế mọi gate đều xuất hiện trong output của
help, nhưng chỉ những gate liên quan mới ảnh hưởng đến hành vi của từng thành phần.

## Cấu hình (Configuration)

### Trong lúc khởi tạo cluster (During cluster initialization)

Tạo một file cấu hình để bật feature gate trên các thành phần liên quan:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
apiServer:
  extraArgs:
    feature-gates: "FeatureName=true"
controllerManager:
  extraArgs:
    feature-gates: "FeatureName=true"
scheduler:
  extraArgs:
    feature-gates: "FeatureName=true"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  FeatureName: true
```

Khởi tạo cluster:

```shell
kubeadm init --config kubeadm-config.yaml
```

### Trên một cluster đang có (On an existing cluster)

Với các cluster kubeadm, cấu hình feature gate có thể được đặt tại nhiều vị trí, bao gồm các
file manifest, các file cấu hình, và cấu hình của kubeadm.

Chỉnh sửa manifest của các thành phần control plane trong `/etc/kubernetes/manifests/`:

1. Với kube-apiserver, kube-controller-manager hoặc kube-scheduler, thêm flag vào phần command:

   ```yaml
   spec:
     containers:
     - command:
       - kube-apiserver
       - --feature-gates=FeatureName=true
       # ... các flag khác
   ```

   Lưu file lại. Pod sẽ tự động khởi động lại.

2. Với kubelet, chỉnh sửa `/var/lib/kubelet/config.yaml`:

   ```yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   featureGates:
     FeatureName: true
   ```

   Khởi động lại kubelet:

   ```shell
   sudo systemctl restart kubelet
   ```

3. Với kube-proxy, chỉnh sửa ConfigMap:

   ```shell
   kubectl -n kube-system edit configmap kube-proxy
   ```

   Thêm feature gate vào phần cấu hình:

   ```yaml
   featureGates:
     FeatureName: true
   ```

   Khởi động lại DaemonSet:

   ```shell
   kubectl -n kube-system rollout restart daemonset kube-proxy
   ```

## Cấu hình nhiều feature gate (Configure multiple feature gates)

Dùng danh sách phân tách bằng dấu phẩy cho các flag dòng lệnh:

```shell
--feature-gates=FeatureA=true,FeatureB=false,FeatureC=true
```

Với các thành phần hỗ trợ file cấu hình (kubelet, kube-proxy):

```yaml
featureGates:
  FeatureA: true
  FeatureB: false
  FeatureC: true
```

> **Ghi chú:** Trong các cluster kubeadm, các thành phần control plane (kube-apiserver,
> kube-controller-manager và kube-scheduler) thường được cấu hình qua flag dòng lệnh trong
> manifest static pod của chúng, đặt tại `/etc/kubernetes/manifests/`. Mặc dù các thành phần
> này có hỗ trợ file cấu hình qua flag `--config`, kubeadm chủ yếu dùng flag dòng lệnh.

## Kiểm chứng cấu hình feature gate (Verify feature gate configuration)

Sau khi cấu hình, hãy kiểm chứng rằng các feature gate đã có hiệu lực. Các phương pháp dưới
đây áp dụng cho các cluster kubeadm, nơi các thành phần control plane chạy dưới dạng
static pod.

### Kiểm tra manifest của các thành phần control plane (Check control plane component manifests)

Xem các feature gate được cấu hình trong manifest static pod:

```shell
kubectl -n kube-system get pod kube-apiserver-<node-name> -o yaml | grep feature-gates
```

### Kiểm tra cấu hình kubelet (Check kubelet configuration)

Dùng endpoint configz của kubelet:

```shell
kubectl proxy --port=8001 &
curl -sSL "http://localhost:8001/api/v1/nodes/<node-name>/proxy/configz" | grep featureGates -A 5
```

Hoặc kiểm tra trực tiếp file cấu hình trên node:

```shell
cat /var/lib/kubelet/config.yaml | grep -A 10 featureGates
```

### Kiểm tra qua endpoint metrics (Check via metrics endpoint)

Trạng thái feature gate được các thành phần Kubernetes công bố dưới dạng metric kiểu
Prometheus (có từ Kubernetes 1.26 trở lên). Truy vấn endpoint metrics để kiểm chứng những
feature gate nào đang bật:

```shell
kubectl get --raw /metrics | grep kubernetes_feature_enabled
```

Để kiểm tra một feature gate cụ thể:

```shell
kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep FeatureName
```

Metric hiển thị `1` cho gate đang bật và `0` cho gate đang tắt.

> **Ghi chú:** Trong các cluster kubeadm, hãy kiểm chứng tất cả các vị trí liên quan nơi
> feature gate có thể được cấu hình, vì cấu hình được phân tán trên nhiều file và nhiều vị trí
> khác nhau.

### Kiểm tra qua endpoint /flagz (Check via /flagz endpoint)

Nếu bạn có quyền truy cập các endpoint gỡ lỗi (debugging endpoint) của một thành phần, và
feature gate `ComponentFlagz` đang được bật cho thành phần đó, bạn có thể xem các flag dòng
lệnh đã được dùng để khởi động thành phần bằng cách truy cập endpoint `/flagz`. Các feature
gate được cấu hình bằng flag dòng lệnh sẽ xuất hiện trong output này.

Endpoint `/flagz` là một phần của *z-pages* trong Kubernetes — các trang cung cấp thông tin
gỡ lỗi lúc chạy (runtime) ở dạng con người đọc được cho các thành phần cốt lõi.

Để biết thêm thông tin, xem
[tài liệu về z-pages](https://kubernetes.io/docs/reference/instrumentation/zpages/).

## Hiểu các yêu cầu riêng theo từng thành phần (Understanding component-specific requirements)

Một vài ví dụ về feature gate gắn với thành phần cụ thể:

- **Thiên về API server**: Các tính năng như `StructuredAuthenticationConfiguration` chủ yếu
  ảnh hưởng đến kube-apiserver
- **Thiên về kubelet**: Các tính năng như `GracefulNodeShutdown` chủ yếu ảnh hưởng đến kubelet
- **Nhiều thành phần**: Một số tính năng đòi hỏi sự phối hợp giữa các thành phần

> **Thận trọng:** Khi một tính năng đòi hỏi nhiều thành phần, bạn phải bật gate trên tất cả
> các thành phần liên quan. Chỉ bật trên một số thành phần có thể dẫn đến hành vi không mong
> đợi hoặc lỗi.

Hãy luôn thử nghiệm feature gate trong môi trường không phải production trước. Các tính năng
Alpha có thể bị gỡ bỏ mà không báo trước.

## Tiếp theo (What's next)

* Đọc [tài liệu tham khảo Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
* Tìm hiểu về [các giai đoạn của tính năng (Feature Stages)](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-stages)
* Xem lại [cấu hình kubeadm](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Một tính năng vừa lên GA ở phiên bản cluster của bạn. Nó có cần bạn bật gate không, và bạn
   có tắt nó đi được không? Tắt thì đánh đổi điều gì?
2. Bạn bật một gate chỉ trên kube-apiserver trong khi tính năng đó cần cả
   kube-controller-manager. Bài cảnh báo điều gì về tình huống này?
3. Trên `k8s-master` của cluster lab, bạn muốn bật `FeatureName=true` cho kube-apiserver đang
   chạy. Bạn sửa file nào, sửa chỗ nào trong file, và có phải tự tay khởi động lại pod không?
4. Cùng là bật một gate trên cluster đang chạy, thao tác cho kubelet và cho kube-proxy khác
   nhau thế nào — từ chỗ sửa cho đến cách làm cấu hình mới có hiệu lực?
5. Chạy `kube-scheduler --help` bạn thấy liệt kê cả một gate vốn chỉ dành cho kubelet. Điều đó
   có nghĩa là scheduler cũng dùng gate đó không? Muốn biết chắc một gate đang bật hay tắt
   trên một thành phần, bài chỉ cách kiểm chứng nào bằng metric?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không cần bật** — các tính năng GA (ổn định) luôn được bật theo mặc định; bạn thường chỉ
   cấu hình gate cho tính năng Alpha hoặc Beta. Với một số gate GA, bạn **vẫn có thể tắt**,
   thường là trong một minor release sau khi lên GA; đánh đổi là **cluster của bạn có thể
   không còn đạt chuẩn tương thích (conformant) của Kubernetes**.
2. Bài đặt hẳn một khối Thận trọng: khi một tính năng đòi hỏi nhiều thành phần, bạn **phải bật
   gate trên tất cả các thành phần liên quan**; chỉ bật trên một số thành phần **có thể dẫn
   đến hành vi không mong đợi hoặc lỗi**.
3. Sửa manifest static pod của kube-apiserver trong **`/etc/kubernetes/manifests/`**: thêm
   dòng `--feature-gates=FeatureName=true` vào danh sách `command` của container. **Không cần
   tự khởi động lại** — với static pod, chỉ cần lưu file là pod tự động khởi động lại.
4. **kubelet**: sửa file **`/var/lib/kubelet/config.yaml`** trên node (mục `featureGates`),
   rồi chạy `sudo systemctl restart kubelet`. **kube-proxy**: sửa **ConfigMap `kube-proxy`**
   trong namespace `kube-system` (`kubectl -n kube-system edit configmap kube-proxy`), rồi
   chạy `kubectl -n kube-system rollout restart daemonset kube-proxy`. Khác biệt cốt lõi: một
   bên là file trên từng node cộng restart service; một bên là đối tượng trong cluster cộng
   rollout restart DaemonSet.
5. **Không.** Tất cả các thành phần Kubernetes dùng chung cùng một bộ định nghĩa feature gate,
   nên mọi gate đều xuất hiện trong output của help — nhưng **chỉ những gate liên quan mới ảnh
   hưởng đến hành vi của từng thành phần**. Trực giác "có trong help nghĩa là có tác dụng" là
   chỗ dễ nhầm. Cách kiểm chứng bằng metric: truy vấn
   `kubectl get --raw /metrics | grep kubernetes_feature_enabled` (lọc thêm tên gate nếu cần);
   metric hiển thị `1` cho gate đang bật và `0` cho gate đang tắt.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
