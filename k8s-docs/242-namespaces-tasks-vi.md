# Chia sẻ một Cluster bằng Namespace (Share a Cluster with Namespaces)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/namespaces/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 1/7 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 25**.

Đây là bài mở đầu giai đoạn 25. Lý thuyết namespace bạn đã đọc ở bài [19](19-namespaces-vi.md)
(nhóm 1b), còn LimitRange và ResourceQuota ở bài [133](133-limit-range-vi.md) và
[134](134-resource-quotas-vi.md) (nhóm 7b, có [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md)).
Trang này là **runbook thao tác**: xem, tạo, xóa namespace và dùng namespace để chia cluster
cho nhiều đội.

**Phải hiểu ở lần đọc này:**

- Bốn namespace ban đầu và vai trò từng cái: `default`, `kube-node-lease` (chứa Lease heartbeat
  của từng node), `kube-public` (mọi người dùng đều đọc được, kể cả người chưa được xác thực —
  bài nói rõ khía cạnh công khai này **chỉ là một quy ước**, không phải yêu cầu bắt buộc),
  `kube-system` (mục *Xem các namespace*).
- `kubectl describe namespaces <name>` in ra **cả hai** thứ: resource quota và limit range. Bài
  phân biệt ngay tại đó — quota theo dõi **mức sử dụng tổng hợp** và đặt giới hạn *cứng* (Hard)
  cho **cả Namespace**; limit range đặt ràng buộc **min/max cho một thực thể đơn lẻ** bên trong
  Namespace.
- Hai phase `Active` và `Terminating`. `kubectl delete namespaces` xóa **mọi thứ** bên trong và
  chạy **bất đồng bộ**; chỉ định một `finalizers` không tồn tại thì namespace vẫn tạo được nhưng
  sẽ **kẹt** ở `Terminating` khi có người xóa nó (mục *Tạo một namespace mới* và *Xóa một
  namespace*).
- Namespace cung cấp đúng hai thứ: một **phạm vi cho tên**, và một **cơ chế để gắn ủy quyền và
  chính sách** vào một phần con của cluster; từ đó mỗi cộng đồng người dùng mới có riêng tài
  nguyên, chính sách và ràng buộc (quota). Bài cũng ghi rõ việc dùng nhiều namespace là **tùy
  chọn** (mục *Hiểu động lực của việc dùng namespace*).
- Bản ghi DNS của Service có dạng `<service-name>.<namespace-name>.svc.cluster.local`. Container
  chỉ dùng `<service-name>` sẽ phân giải tới Service **cùng namespace**; muốn truy cập xuyên
  namespace phải dùng FQDN (mục *Hiểu về namespace và DNS*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cơ chế `finalizers` và link tài liệu thiết kế của nó trong mục *Tạo một namespace mới* | ở đây chỉ cần biết finalizer sai làm namespace kẹt `Terminating`, chưa cần biết finalizer chạy thế nào | bài [29 — Finalizers](29-finalizers-vi.md), nhóm [1c của giai đoạn 1](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object) — đã đọc |
| Link *Admission control: Limit Range* (tài liệu thiết kế ngoài kubernetes.io) | là đề xuất thiết kế cũ, không phải cách cấu hình LimitRange hôm nay | các bài con của [228](228-manage-resources-tasks-vi.md) — bài 2/7, ngay sau bài này |
| Đoạn cuối mục *Tạo pod trong mỗi namespace* nói kịch bản sẽ mở rộng sang các quy tắc ủy quyền khác nhau cho từng namespace | cách **viết** quy tắc ủy quyền theo namespace không nằm trong bài này | RBAC ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) và [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) — đã học |

---

Trang này trình bày cách xem, làm việc bên trong và xóa các namespace. Trang cũng trình bày cách
dùng namespace của Kubernetes để phân chia cluster của bạn.

## Trước khi bạn bắt đầu (Before you begin)

* Có sẵn một [cluster Kubernetes](https://kubernetes.io/docs/setup/).
* Bạn có hiểu biết cơ bản về Pod, Service và Deployment trong Kubernetes.

## Xem các namespace (Viewing namespaces)

Liệt kê các namespace hiện có trong một cluster bằng:

```shell
kubectl get namespaces
```
```console
NAME              STATUS   AGE
default           Active   11d
kube-node-lease   Active   11d
kube-public       Active   11d
kube-system       Active   11d
```

Kubernetes khởi đầu với bốn namespace ban đầu:

* `default` Namespace mặc định cho các đối tượng không thuộc namespace nào khác
* `kube-node-lease` Namespace này chứa các đối tượng
  [Lease](35-leases-vi.md) gắn với từng node. Node
  lease cho phép kubelet gửi các
  [heartbeat](https://kubernetes.io/docs/concepts/architecture/nodes#heartbeats) để control
  plane có thể phát hiện node bị lỗi.
* `kube-public` Namespace này được tạo tự động và mọi người dùng đều đọc được (kể cả những
  người chưa được xác thực). Namespace này chủ yếu được dành riêng cho việc sử dụng của cluster,
  trong trường hợp một số tài nguyên cần được hiển thị và đọc được công khai trong toàn bộ
  cluster. Khía cạnh công khai của namespace này chỉ là một quy ước, không phải một yêu cầu
  bắt buộc.
* `kube-system` Namespace cho các đối tượng do hệ thống Kubernetes tạo ra

Bạn cũng có thể lấy thông tin tóm tắt của một namespace cụ thể bằng:

```shell
kubectl get namespaces <name>
```

Hoặc bạn có thể lấy thông tin chi tiết với:

```shell
kubectl describe namespaces <name>
```
```console
Name:           default
Labels:         <none>
Annotations:    <none>
Status:         Active

No resource quota.

Resource Limits
 Type       Resource    Min Max Default
 ----               --------    --- --- ---
 Container          cpu         -   -   100m
```

Lưu ý rằng các chi tiết này hiển thị cả resource quota (nếu có) lẫn các resource limit range.

Resource quota theo dõi mức sử dụng tổng hợp các tài nguyên trong Namespace và cho phép người
vận hành cluster định nghĩa các giới hạn sử dụng tài nguyên *cứng* (Hard) mà một Namespace được
phép tiêu thụ.

Một limit range định nghĩa các ràng buộc min/max về lượng tài nguyên mà một thực thể đơn lẻ có
thể tiêu thụ trong một Namespace.

Xem [Admission control: Limit Range](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_limit_range.md)

Một namespace có thể ở một trong hai giai đoạn (phase):

* `Active` namespace đang được sử dụng
* `Terminating` namespace đang được xóa và không thể dùng cho các đối tượng mới

Để biết thêm chi tiết, xem
[Namespace](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/namespace-v1/)
trong tài liệu tham chiếu API.

## Tạo một namespace mới (Creating a new namespace) {#creating-a-new-namespace}

> **Ghi chú:**
> Tránh tạo namespace có tiền tố `kube-`, vì tiền tố này được dành riêng cho các namespace
> hệ thống của Kubernetes.

Tạo một file YAML mới tên là `my-namespace.yaml` với nội dung:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here>
```

Sau đó chạy:

```shell
kubectl create -f ./my-namespace.yaml
```

Cách khác, bạn có thể tạo namespace bằng lệnh dưới đây:

```shell
kubectl create namespace <insert-namespace-name-here>
```

Tên namespace của bạn phải là một
[nhãn DNS (DNS label)](17-names-vi.md#dns-label-names)
hợp lệ.

Có một trường tùy chọn là `finalizers`, cho phép các bên quan sát (observables) dọn sạch tài
nguyên mỗi khi namespace bị xóa. Hãy nhớ rằng nếu bạn chỉ định một finalizer không tồn tại,
namespace vẫn sẽ được tạo nhưng sẽ bị kẹt ở trạng thái `Terminating` nếu người dùng cố xóa nó.

Bạn có thể tìm thêm thông tin về `finalizers` trong
[tài liệu thiết kế](https://git.k8s.io/design-proposals-archive/architecture/namespaces.md#finalizers)
của namespace.

## Xóa một namespace (Deleting a namespace) {#deleting-a-namespace}

Xóa một namespace bằng

```shell
kubectl delete namespaces <insert-some-namespace-name>
```

> **Cảnh báo:**
> Thao tác này xóa _mọi thứ_ bên trong namespace!

Việc xóa này diễn ra bất đồng bộ (asynchronous), vì vậy trong một khoảng thời gian bạn sẽ thấy
namespace ở trạng thái `Terminating`.

## Phân chia cluster của bạn bằng namespace của Kubernetes (Subdividing your cluster using Kubernetes namespaces)

Theo mặc định, một cluster Kubernetes sẽ khởi tạo một namespace default khi cấp phát (provision)
cluster để chứa tập hợp mặc định các Pod, Service và Deployment mà cluster sử dụng.

Giả sử bạn có một cluster mới, bạn có thể xem các namespace hiện có bằng cách làm như sau:

```shell
kubectl get namespaces
```
```console
NAME      STATUS    AGE
default   Active    13m
```

### Tạo các namespace mới (Create new namespaces)

Trong bài tập này, bạn tạo thêm hai namespace Kubernetes để chứa nội dung của bạn.

Trong một kịch bản mà một tổ chức đang dùng chung một cluster Kubernetes cho cả mục đích phát
triển (development) và sản xuất (production):

- Đội phát triển muốn duy trì một không gian trong cluster nơi họ có thể xem danh sách các Pod,
  Service và Deployment mà họ dùng để xây dựng và chạy ứng dụng của mình. Trong không gian này,
  các tài nguyên Kubernetes được tạo ra và mất đi liên tục, và các hạn chế về việc ai được hay
  không được sửa đổi tài nguyên được nới lỏng để hỗ trợ phát triển linh hoạt (agile).

- Đội vận hành muốn duy trì một không gian trong cluster nơi họ có thể áp đặt các quy trình
  nghiêm ngặt về việc ai được hay không được thao tác trên tập hợp các Pod, Service và
  Deployment đang chạy trang production.

Một khuôn mẫu mà tổ chức này có thể áp dụng là phân vùng cluster Kubernetes thành hai namespace:
`development` và `production`. Hãy tạo hai namespace mới để chứa phần việc của bạn.

Tạo namespace `development` bằng kubectl:

```shell
kubectl create -f https://k8s.io/examples/admin/namespace-dev.json
```

Tạo namespace `production` bằng kubectl:

```shell
kubectl create -f https://k8s.io/examples/admin/namespace-prod.json
```

Để chắc chắn mọi thứ đã đúng, liệt kê tất cả các namespace trong cluster.

```shell
kubectl get namespaces --show-labels
```

```console
NAME          STATUS    AGE       LABELS
default       Active    32m       <none>
development   Active    29s       name=development
production    Active    23s       name=production
```

### Tạo pod trong mỗi namespace (Create pods in each namespace)

Một namespace của Kubernetes cung cấp phạm vi (scope) cho các Pod, Service và Deployment trong
cluster. Người dùng tương tác với một namespace sẽ không nhìn thấy nội dung trong namespace
khác. Để minh họa điều này, hãy tạo một Deployment và các Pod trong namespace `development`.

```shell
kubectl create deployment snowflake \
  --image=registry.k8s.io/serve_hostname \
  -n=development --replicas=2
```

Bạn vừa tạo một deployment với số replica là 2, chạy pod tên là `snowflake` với một container cơ
bản phục vụ trả về hostname.

```shell
kubectl get deployment -n=development
```
```console
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
snowflake    2/2     2            2           2m
```

```shell
kubectl get pods -l app=snowflake -n=development
```
```console
NAME                         READY     STATUS    RESTARTS   AGE
snowflake-3968820950-9dgr8   1/1       Running   0          2m
snowflake-3968820950-vgc4n   1/1       Running   0          2m
```

Điều này cho thấy các nhà phát triển có thể làm những gì họ muốn mà không phải lo lắng về việc
ảnh hưởng tới nội dung trong namespace `production`.

Chuyển sang namespace `production` và xem cách tài nguyên trong một namespace bị ẩn khỏi
namespace kia. Namespace `production` lúc này còn trống, và các lệnh sau sẽ không trả về gì.

```shell
kubectl get deployment -n=production
kubectl get pods -n=production
```

Tạo một số pod trong namespace `production`.

```shell
kubectl create deployment cattle --image=registry.k8s.io/serve_hostname -n=production
kubectl scale deployment cattle --replicas=5 -n=production

kubectl get deployment -n=production
```

```console
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
cattle       5/5     5            5           10s
```

```shell
kubectl get pods -l app=cattle -n=production
```
```console
NAME                      READY     STATUS    RESTARTS   AGE
cattle-2263376956-41xy6   1/1       Running   0          34s
cattle-2263376956-kw466   1/1       Running   0          34s
cattle-2263376956-n4v97   1/1       Running   0          34s
cattle-2263376956-p5p3i   1/1       Running   0          34s
cattle-2263376956-sxpth   1/1       Running   0          34s
```

Đến đây, hẳn đã rõ ràng rằng các tài nguyên mà người dùng tạo trong một namespace bị ẩn khỏi
namespace kia.

Khi khả năng hỗ trợ chính sách (policy) trong Kubernetes phát triển thêm, kịch bản này sẽ được
mở rộng để cho thấy cách bạn có thể cung cấp các quy tắc ủy quyền (authorization) khác nhau cho
từng namespace.

## Hiểu động lực của việc dùng namespace (Understanding the motivation for using namespaces)

Một cluster đơn lẻ nên có khả năng đáp ứng nhu cầu của nhiều người dùng hoặc nhiều nhóm người
dùng (từ đây trong tài liệu này gọi là một _cộng đồng người dùng_).

Các _namespace_ của Kubernetes giúp các dự án, các đội nhóm hoặc các khách hàng khác nhau chia
sẻ chung một cluster Kubernetes.

Nó làm được điều đó bằng cách cung cấp:

1. Một phạm vi cho các [tên (names)](17-names-vi.md).
1. Một cơ chế để gắn việc ủy quyền và chính sách vào một phần con của cluster.

Việc dùng nhiều namespace là tùy chọn.

Mỗi cộng đồng người dùng muốn có thể làm việc tách biệt với các cộng đồng khác. Mỗi cộng đồng
người dùng có riêng:

1. các tài nguyên (pod, service, replication controller, v.v.)
1. các chính sách (ai được hay không được thực hiện hành động trong cộng đồng của họ)
1. các ràng buộc (cộng đồng này được cấp chừng này quota, v.v.)

Người vận hành cluster có thể tạo một Namespace cho mỗi cộng đồng người dùng riêng biệt.

Namespace cung cấp một phạm vi duy nhất cho:

1. các tài nguyên có tên (để tránh xung đột tên cơ bản)
1. việc ủy thác quyền quản lý cho những người dùng được tin cậy
1. khả năng giới hạn mức tiêu thụ tài nguyên của cộng đồng

Các trường hợp sử dụng bao gồm:

1. Với vai trò người vận hành cluster, tôi muốn hỗ trợ nhiều cộng đồng người dùng trên một
   cluster duy nhất.
1. Với vai trò người vận hành cluster, tôi muốn ủy thác quyền quản lý các phân vùng của cluster
   cho những người dùng được tin cậy trong các cộng đồng đó.
1. Với vai trò người vận hành cluster, tôi muốn giới hạn lượng tài nguyên mỗi cộng đồng có thể
   tiêu thụ nhằm hạn chế ảnh hưởng tới các cộng đồng khác đang dùng cluster.
1. Với vai trò người dùng cluster, tôi muốn tương tác với các tài nguyên liên quan tới cộng
   đồng người dùng của mình một cách tách biệt khỏi những gì các cộng đồng người dùng khác đang
   làm trên cluster.

## Hiểu về namespace và DNS (Understanding namespaces and DNS)

Khi bạn tạo một [Service](82-service-vi.md), nó
tạo một [bản ghi DNS (DNS entry)](10-dns-pod-service-vi.md)
tương ứng. Bản ghi này có dạng `<service-name>.<namespace-name>.svc.cluster.local`, nghĩa là
nếu một container chỉ dùng `<service-name>`, nó sẽ phân giải tới service nằm cùng namespace.
Điều này hữu ích khi dùng cùng một cấu hình cho nhiều namespace như Development, Staging và
Production. Nếu bạn muốn truy cập xuyên namespace, bạn cần dùng tên miền đầy đủ (fully qualified
domain name — FQDN).

## Tiếp theo (What's next)

* Tìm hiểu thêm về [thiết lập namespace ưa dùng](19-namespaces-vi.md#setting-the-namespace-preference).
* Tìm hiểu thêm về [thiết lập namespace cho một request](19-namespaces-vi.md#setting-the-namespace-for-a-request)
* Xem [thiết kế namespace](https://git.k8s.io/design-proposals-archive/architecture/namespaces.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. Trên `lab-k8s-master`, bạn tạo hai namespace `development` và `production`, đặt Deployment
   trong cả hai, rồi gõ `kubectl get pods` **không kèm `-n`**. Kết quả trống. Lệnh đó đang hỏi
   namespace nào, và trong bốn namespace ban đầu thì namespace nào chứa Lease heartbeat của
   `lab-k8s-worker1` và `lab-k8s-worker2`?
2. **Câu bẫy.** `kubectl describe namespaces default` in ra `No resource quota.` rồi ngay bên
   dưới là một mục `Resource Limits` với dòng `Container cpu - - 100m`. Hai khối đó nói về hai
   thứ khác nhau — mỗi khối đặt ràng buộc cho **cái gì**? Vì sao "chưa có quota" không có nghĩa
   là "container muốn xin bao nhiêu CPU cũng được"?
3. Một Pod trong namespace `development` gọi `http://web` trong khi Service `web` nằm ở namespace
   `production`. Theo cơ chế DNS của bài, tên đó phân giải tới đâu, và phải viết địa chỉ thế nào
   cho đúng?
4. Bạn chạy `kubectl delete namespaces development`, lệnh trả về ngay nhưng `kubectl get
   namespaces` vẫn thấy nó ở `Terminating` một lúc. Vì sao đó là hành vi bình thường — và trường
   hợp nào namespace **không bao giờ** thoát khỏi `Terminating`?
5. Bài kết luận rằng tài nguyên người dùng tạo trong namespace này "bị ẩn khỏi" namespace kia.
   Theo đúng lập luận của bài, "ẩn" đến từ cơ chế nào — và bản thân việc tạo namespace đã đủ để
   chặn đội `development` thao tác lên `production` chưa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó đang hỏi namespace **`default`** — namespace mặc định cho các đối tượng không thuộc
   namespace nào khác; Pod của bạn nằm ở `development` và `production` nên không xuất hiện. Lease
   heartbeat của hai worker nằm ở **`kube-node-lease`**: namespace này chứa các đối tượng Lease
   gắn với từng node, để kubelet gửi heartbeat cho control plane phát hiện node bị lỗi.
2. **Quota đặt trần cho cả Namespace, limit range đặt ràng buộc cho từng thực thể đơn lẻ.** Khối
   quota theo dõi *mức sử dụng tổng hợp* các tài nguyên trong Namespace và định nghĩa giới hạn
   *cứng* (Hard) mà cả Namespace được phép tiêu thụ; `No resource quota.` chỉ nói rằng trần tổng
   đó chưa được đặt. Khối `Resource Limits` là limit range — nó định nghĩa ràng buộc min/max về
   lượng tài nguyên mà **một thực thể đơn lẻ** có thể tiêu thụ, và dòng ví dụ còn đang đặt giá
   trị **Default** `100m` CPU cho mỗi container. Nên "chưa có quota" **không** đồng nghĩa với
   "không có ràng buộc": đây là hai cơ chế độc lập, `describe` chỉ tiện in cả hai cùng một chỗ.
3. Nó phân giải tới Service **tên `web` nằm trong chính namespace `development`** — không phải
   Service ở `production`. Bản ghi DNS có dạng `<service-name>.<namespace-name>.svc.cluster.local`,
   nên tên ngắn luôn được hiểu là service cùng namespace. Muốn truy cập xuyên namespace phải dùng
   **FQDN**: `web.production.svc.cluster.local`.
4. Vì **việc xóa namespace diễn ra bất đồng bộ**: lệnh trả về trước khi mọi thứ bên trong bị dọn
   xong, nên trong một khoảng thời gian namespace ở phase `Terminating` và không dùng được cho
   đối tượng mới. Trường hợp kẹt vĩnh viễn: namespace được tạo kèm một **`finalizers` không tồn
   tại** — namespace vẫn được tạo, nhưng khi có người xóa thì nó **kẹt ở `Terminating`**.
5. "Ẩn" đến từ việc namespace là một **phạm vi (scope)** cho Pod, Service và Deployment: người
   dùng tương tác với một namespace không nhìn thấy nội dung trong namespace khác, đúng như minh
   họa `snowflake` ở `development` và `cattle` ở `production`. Nhưng **chưa đủ**: bài nói namespace
   cung cấp *hai* thứ — phạm vi cho tên **và** một cơ chế để gắn ủy quyền và chính sách vào một
   phần con của cluster. Phần ủy quyền phải được cấu hình riêng; chính bài cũng ghi rằng kịch bản
   này còn phải mở rộng thì mới cung cấp được các quy tắc ủy quyền khác nhau cho từng namespace.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
