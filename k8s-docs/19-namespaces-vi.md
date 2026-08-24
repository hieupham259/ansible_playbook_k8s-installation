# Namespaces

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/>
>
> Trong Kubernetes, namespace cung cấp cơ chế cô lập các nhóm resource bên trong
> một cluster duy nhất.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 4/9 · Kiểm chứng ở [Lab 1b](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md).

Lab 1a đã dùng namespace như một vùng chứa tạm mà không giải thích. Đây là chỗ trả nợ đó.

**Phải hiểu ở lần đọc này:**

- Namespace là **phạm vi cho tên**, không phải ranh giới bảo mật tự động. Muốn cô lập thật thì
  phải cộng thêm RBAC, quota và NetworkPolicy — những thứ học sau.
- **Resource namespaced và resource cấp cluster** là hai loại tách bạch. Node,
  PersistentVolume, StorageClass và chính Namespace đều **không** thuộc namespace nào. Kiểm
  bằng `kubectl api-resources --namespaced=true` và `--namespaced=false`.
- Bốn namespace ban đầu và vai trò từng cái — đặc biệt `kube-node-lease`, nơi bạn đã quan sát
  heartbeat ở Lab 1a phần B6.
- Namespace **không lồng nhau**; mỗi resource nằm trong đúng một namespace.
- Đừng dùng namespace để phân biệt thứ chỉ hơi khác nhau (các version của cùng một app) —
  việc đó là của label.
- Tên namespace phải là **DNS label theo RFC 1123**, tức chặt hơn tên object thông thường.
- `--namespace` cho một lệnh, `kubectl config set-context --current --namespace=...` cho cả
  context.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Namespace và DNS*, dạng FQDN `svc.cluster.local` | chưa học Service và DNS trong cluster | giai đoạn 5 |
| Cảnh báo về namespace trùng tên TLD công cộng | hệ quả nằm ở tầng phân giải DNS | giai đoạn 5 và 9 |
| Resource quota để chia tài nguyên | là chủ đề chính sách riêng | giai đoạn 7 |

---

Trong Kubernetes, _namespace_ cung cấp một cơ chế để cô lập (isolate) các nhóm resource bên trong một cluster duy nhất. Tên của các resource phải là duy nhất trong một namespace, nhưng không cần duy nhất giữa các namespace. Phạm vi (scope) theo namespace chỉ áp dụng cho các object thuộc namespace _(ví dụ: Deployment, Service, v.v.)_ chứ không áp dụng cho các object ở phạm vi toàn cluster _(ví dụ: StorageClass, Node, PersistentVolume, v.v.)_.

## Khi nào nên dùng nhiều namespace (When to Use Multiple Namespaces)

Namespace được thiết kế để dùng trong các môi trường có nhiều người dùng trải rộng trên nhiều
team hoặc nhiều dự án. Với các cluster chỉ có từ vài đến vài chục người dùng, bạn hoàn toàn không
cần phải tạo hay bận tâm đến namespace. Hãy bắt đầu dùng namespace khi bạn
cần đến những tính năng mà chúng cung cấp.

Namespace cung cấp một phạm vi cho tên (scope for names). Tên của các resource phải là duy nhất trong một namespace,
nhưng không cần duy nhất giữa các namespace. Các namespace không thể lồng vào nhau và mỗi resource
Kubernetes chỉ có thể nằm trong một namespace.

Namespace là một cách để phân chia tài nguyên cluster giữa nhiều người dùng (thông qua [resource quota](134-resource-quotas-vi.md)).

Không nhất thiết phải dùng nhiều namespace để tách các resource chỉ hơi khác nhau,
chẳng hạn các phiên bản khác nhau của cùng một phần mềm: hãy dùng
[label](./18-labels-vi.md) để phân biệt
các resource trong cùng một namespace.

> **Ghi chú:**
> Với cluster production, hãy cân nhắc _không_ dùng namespace `default`. Thay vào đó, hãy tạo các namespace khác và sử dụng chúng.

## Các namespace ban đầu (Initial namespaces)

Kubernetes khởi đầu với bốn namespace ban đầu:

- `default`: Kubernetes bao gồm namespace này để bạn có thể bắt đầu sử dụng cluster mới của mình mà không cần tạo namespace trước.

- `kube-node-lease`: Namespace này chứa các object [Lease](35-leases-vi.md) gắn với từng node. Node lease cho phép kubelet gửi [heartbeat](23-nodes-vi.md#node-heartbeats) để control plane có thể phát hiện node bị lỗi.

- `kube-public`: Namespace này có thể được đọc bởi *tất cả* các client (kể cả những client chưa xác thực). Namespace này chủ yếu được dành cho mục đích sử dụng của cluster, trong trường hợp một số resource cần được hiển thị và đọc công khai trên toàn cluster. Khía cạnh công khai của namespace này chỉ là một quy ước, không phải là một yêu cầu bắt buộc.

- `kube-system`: Namespace dành cho các object do hệ thống Kubernetes tạo ra.

## Làm việc với Namespace (Working with Namespaces)

Việc tạo và xóa namespace được mô tả trong
[tài liệu Hướng dẫn quản trị về namespace](242-namespaces-tasks-vi.md).

> **Ghi chú:**
> Tránh tạo namespace có prefix `kube-`, vì prefix này được dành riêng cho các namespace hệ thống của Kubernetes.

### Xem các namespace (Viewing namespaces)

Bạn có thể liệt kê các namespace hiện có trong cluster bằng lệnh:

```shell
kubectl get namespace
```
```
NAME              STATUS   AGE
default           Active   1d
kube-node-lease   Active   1d
kube-public       Active   1d
kube-system       Active   1d
```

### Đặt namespace cho một request (Setting the namespace for a request) {#setting-the-namespace-for-a-request}

Để đặt namespace cho request hiện tại, dùng cờ (flag) `--namespace`.

Ví dụ:

```shell
kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>
kubectl get pods --namespace=<insert-namespace-name-here>
```

### Đặt namespace mặc định ưu tiên (Setting the namespace preference) {#setting-the-namespace-preference}

Bạn có thể lưu vĩnh viễn namespace cho tất cả các lệnh kubectl tiếp theo trong
context đó.

```shell
kubectl config set-context --current --namespace=<insert-namespace-name-here>
# Kiểm tra lại
kubectl config view --minify | grep namespace:
```

## Namespace và DNS (Namespaces and DNS)

Khi bạn tạo một [Service](82-service-vi.md),
nó sẽ tạo một [bản ghi DNS](./10-dns-pod-service-vi.md) tương ứng.
Bản ghi này có dạng `<service-name>.<namespace-name>.svc.cluster.local`, nghĩa là
nếu một container chỉ dùng `<service-name>`, nó sẽ phân giải (resolve) tới service
cục bộ trong namespace đó. Điều này hữu ích khi dùng cùng một cấu hình trên
nhiều namespace, chẳng hạn Development, Staging và Production. Nếu bạn muốn truy cập
xuyên namespace, bạn cần dùng tên miền đầy đủ (fully qualified domain name — FQDN).

Do đó, tất cả tên namespace phải là
[DNS label hợp lệ theo RFC 1123](17-names-vi.md#dns-label-names).

> **Cảnh báo:**
> Bằng cách tạo các namespace trùng tên với [các tên miền cấp cao nhất (top-level domain) công cộng](https://data.iana.org/TLD/tlds-alpha-by-domain.txt), các Service trong những
> namespace này có thể có tên DNS ngắn trùng lặp với các bản ghi DNS công cộng.
> Workload từ bất kỳ namespace nào khi thực hiện tra cứu DNS mà không có [dấu chấm ở cuối (trailing dot)](https://datatracker.ietf.org/doc/html/rfc1034#page-8) sẽ
> bị chuyển hướng tới các service đó, được ưu tiên hơn DNS công cộng.
>
> Để giảm thiểu rủi ro này, hãy giới hạn quyền tạo namespace cho những người dùng đáng tin cậy. Nếu
> cần, bạn cũng có thể cấu hình thêm các cơ chế kiểm soát bảo mật bên thứ ba, chẳng hạn
> [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/),
> để chặn việc tạo bất kỳ namespace nào trùng tên với [các TLD công cộng](https://data.iana.org/TLD/tlds-alpha-by-domain.txt).

## Không phải object nào cũng nằm trong namespace (Not all objects are in a namespace)

Hầu hết các resource của Kubernetes (ví dụ: Pod, Service, Deployment và các loại khác) đều nằm trong một namespace nào đó. Tuy nhiên, bản thân resource namespace lại không nằm trong một namespace. Và các resource cấp thấp, chẳng hạn
[Node](23-nodes-vi.md) và
[PersistentVolume](92-persistent-volumes-vi.md), không nằm trong bất kỳ namespace nào.

Để xem những resource Kubernetes nào nằm trong và không nằm trong namespace:

```shell
# Nằm trong namespace
kubectl api-resources --namespaced=true

# Không nằm trong namespace
kubectl api-resources --namespaced=false
```

## Gán nhãn tự động (Automatic labelling)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [stable]`

Control plane của Kubernetes đặt một label bất biến (immutable)
`kubernetes.io/metadata.name` trên tất cả các namespace.
Giá trị của label này là tên của namespace.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [tạo một namespace mới](242-namespaces-tasks-vi.md#creating-a-new-namespace).
* Tìm hiểu thêm về [xóa một namespace](242-namespaces-tasks-vi.md#deleting-a-namespace).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Node và PersistentVolume có nằm trong namespace không? Bạn kiểm chứng bằng lệnh nào trên
   cluster lab?
2. Ở Lab 1a bạn theo dõi Lease trong `kube-node-lease`. Namespace đó chứa gì, và thành phần
   nào ghi vào đó?
3. Bạn có ba phiên bản của cùng một ứng dụng chạy song song. Nên tách ba namespace hay dùng
   label? Vì sao?
4. Tạo được namespace tên `My_App` không? Nếu không thì vì sao, và ràng buộc đó chặt hơn tên
   object thông thường ở chỗ nào?
5. Đặt một Pod vào hai namespace cùng lúc có được không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Cả hai đều là resource cấp cluster. Kiểm bằng
   `kubectl api-resources --namespaced=false` — Node, PersistentVolume, StorageClass và chính
   Namespace đều xuất hiện ở đó.
2. Chứa object **Lease** gắn với từng Node. **Kubelet** trên mỗi node ghi vào Lease của node
   mình để gửi heartbeat, còn node controller đọc để phát hiện node mất liên lạc.
3. **Dùng label.** Bài nói rõ: không cần dùng nhiều namespace để tách các resource chỉ hơi
   khác nhau như các phiên bản của cùng một phần mềm. Namespace dành cho nhiều team hoặc nhiều
   dự án, và nó kéo theo cả quota lẫn phân quyền.
4. **Không.** Tên namespace phải là DNS label theo RFC 1123: chỉ chữ-số **thường** và `-`, tối
   đa 63 ký tự. `My_App` sai cả vì chữ hoa lẫn vì dấu gạch dưới. Tên object thông thường
   thường chỉ cần là DNS subdomain — cho phép dấu chấm và dài tới 253 ký tự.
5. **Không.** Namespace không lồng nhau và mỗi resource chỉ nằm trong đúng một namespace.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
