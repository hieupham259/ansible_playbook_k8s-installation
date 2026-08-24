# Tên và ID của đối tượng (Object Names and IDs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/names/>
>
> Mỗi đối tượng trong cluster của bạn có một Tên (Name) duy nhất cho loại tài nguyên đó.
> Mỗi đối tượng Kubernetes cũng có một UID duy nhất trên toàn bộ cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 1/9 · Kiểm chứng ở [Lab 1b](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md).

**Phải hiểu ở lần đọc này:**

- Kubernetes định danh một object bằng **bốn thuộc tính**: API group, resource type, namespace,
  name. **Phiên bản API không nằm trong đó** — không thể lách trùng tên bằng cách dùng version
  khác.
- Name duy nhất trong phạm vi đó **tại một thời điểm**; xóa rồi tạo lại trùng tên là hợp lệ.
- UID do hệ thống sinh, duy nhất toàn cluster và **phân biệt các lần xuất hiện trong lịch sử**
  của cùng một tên. Bạn đã dùng chính điều này ở Lab 1a phần B8.
- Ràng buộc **DNS subdomain** (≤253 ký tự, chữ-số thường, `-` và `.`) — dạng phổ biến nhất.
- `generateName`: server tự nối hậu tố vào tiền tố bạn đưa.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khác biệt chi tiết giữa RFC 1123 và RFC 1035 label | chỉ cần khi gặp lỗi validation cụ thể | tra lại khi bị từ chối tên |
| Feature gate `RelaxedServiceNameValidation` | liên quan tới Service | giai đoạn 5 |
| *Tên dùng làm phân đoạn đường dẫn* | ràng buộc hiếm gặp | tra khi cần |

Đừng học thuộc bốn bộ ràng buộc. Chỉ cần biết **chúng khác nhau** và khi API server từ chối
một cái tên thì phải tra xem resource đó đòi dạng nào.

Đọc xong trang gốc thì đọc tiếp
[Bốn thuộc tính định danh trong thực tế](#bon-thuoc-tinh-trong-thuc-te) ở cuối bài — phần bổ
sung diễn giải **API group** bằng ví dụ triển khai thật (manifest, RBAC, CRD, nâng cấp cluster).

---

Mỗi đối tượng trong cluster của bạn có một [_Tên (Name)_](#names) duy nhất cho loại tài nguyên (resource) đó.
Mỗi đối tượng Kubernetes cũng có một [_UID_](#uids) duy nhất trên toàn bộ cluster của bạn.

Ví dụ, bạn chỉ có thể có một Pod tên `myapp-1234` trong cùng một [namespace](19-namespaces-vi.md), nhưng bạn có thể có một Pod và một Deployment cùng mang tên `myapp-1234`.

Với các thuộc tính do người dùng cung cấp mà không cần duy nhất, Kubernetes cung cấp [label](18-labels-vi.md) và [annotation](20-annotations-vi.md).

## Tên (Names) {#names}

Một chuỗi do client cung cấp dùng để tham chiếu tới một đối tượng trong URL tài nguyên (resource URL), chẳng hạn `/api/v1/pods/some-name`.

Tại một thời điểm, chỉ một đối tượng của một loại (kind) nhất định được mang một tên nhất định. Tuy nhiên, nếu bạn xóa đối tượng đó, bạn có thể tạo một đối tượng mới trùng tên.

Tên phải là duy nhất trên tất cả các [phiên bản API](21-kubernetes-api-vi.md#api-groups-and-versioning) của cùng một tài nguyên.

Kubernetes định danh duy nhất các đối tượng bằng tổ hợp của bốn thuộc tính:
* **Nhóm API (API group)** (ví dụ: `apps`)
* **Loại tài nguyên (Resource type)** (ví dụ: `deployments`)
* **Namespace** (đối với các tài nguyên thuộc namespace)
* **Tên (Name)**

Mặc dù bạn có thể truy cập một tài nguyên qua các phiên bản API khác nhau (chẳng hạn `v1` hoặc `v1beta1`), phiên bản chỉ đơn giản là một cách biểu diễn khác của cùng một đối tượng bên dưới. Vì phiên bản không phải là một phần của định danh duy nhất, bạn không thể tạo hai đối tượng có cùng tên và cùng loại tài nguyên trong cùng một namespace bằng cách dùng các phiên bản API khác nhau.

> **Ghi chú:** Trong những trường hợp đối tượng đại diện cho một thực thể vật lý, như một Node đại diện cho một máy chủ (host) vật lý, khi máy chủ được tạo lại với cùng tên mà Node không bị xóa đi và tạo lại, Kubernetes sẽ coi máy chủ mới là máy chủ cũ, điều này có thể dẫn đến những điểm không nhất quán.

Server có thể sinh tên khi `generateName` được cung cấp thay cho `name` trong yêu cầu tạo tài nguyên.
Khi dùng `generateName`, giá trị được cung cấp sẽ được dùng làm tiền tố (prefix) của tên, và server sẽ nối thêm
một hậu tố (suffix) được sinh ra vào đó. Dù tên được sinh tự động, nó vẫn có thể xung đột với các tên đã tồn tại,
dẫn đến phản hồi HTTP 409. Điều này trở nên ít có khả năng xảy ra hơn nhiều trong Kubernetes v1.31 trở đi,
vì server sẽ thực hiện tối đa 8 lần thử sinh một tên duy nhất trước khi trả về phản hồi HTTP 409.

Dưới đây là bốn loại ràng buộc tên (name constraint) thường dùng cho các tài nguyên.

### Tên DNS Subdomain (DNS Subdomain Names) {#dns-subdomain-names}

Hầu hết các loại tài nguyên yêu cầu tên có thể dùng làm tên DNS subdomain
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa không quá 253 ký tự
- chỉ chứa các ký tự chữ và số viết thường (lowercase alphanumeric), '-' hoặc '.'
- bắt đầu bằng một ký tự chữ hoặc số
- kết thúc bằng một ký tự chữ hoặc số

### Tên label theo RFC 1123 (RFC 1123 Label Names) {#dns-label-names}

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Khi feature gate `RelaxedServiceNameValidation` được bật,
> tên các đối tượng Service được phép bắt đầu bằng chữ số.

### Tên label theo RFC 1035 (RFC 1035 Label Names)

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1035](https://tools.ietf.org/html/rfc1035).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Mặc dù về mặt kỹ thuật RFC 1123 cho phép label bắt đầu bằng chữ số, hiện thực (implementation)
> hiện tại của Kubernetes yêu cầu cả label theo RFC 1035 lẫn RFC 1123 đều phải bắt đầu
> bằng một ký tự chữ cái. Ngoại lệ là khi feature gate `RelaxedServiceNameValidation`
> được bật cho các đối tượng Service, khi đó tên Service được phép bắt đầu bằng chữ số.

### Tên dùng làm phân đoạn đường dẫn (Path Segment Names)

Một số loại tài nguyên yêu cầu tên của chúng có thể được mã hóa an toàn thành một
phân đoạn đường dẫn (path segment). Nói cách khác, tên không được là "." hay ".." và tên
không được chứa "/" hay "%".

Dưới đây là một manifest ví dụ cho một Pod tên `nginx-demo`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

> **Ghi chú:** Một số loại tài nguyên có thêm các ràng buộc bổ sung đối với tên của chúng.

## Các UID (UIDs) {#uids}

Một chuỗi do hệ thống Kubernetes sinh ra để định danh duy nhất các đối tượng.

Mỗi đối tượng được tạo ra trong suốt vòng đời của một cluster Kubernetes đều có một UID riêng biệt. UID nhằm mục đích phân biệt giữa các lần xuất hiện trong lịch sử của những thực thể tương tự nhau.

UID của Kubernetes là các định danh duy nhất toàn cầu (universally unique identifiers, còn được gọi là UUID).
UUID được chuẩn hóa theo ISO/IEC 9834-8 và ITU-T X.667.

## Tiếp theo (What's next)

* Đọc về [label](18-labels-vi.md) và [annotation](20-annotations-vi.md) trong Kubernetes.
* Xem tài liệu thiết kế [Identifiers and Names in Kubernetes](https://git.k8s.io/design-proposals-archive/architecture/identifiers.md).

---

## Bốn thuộc tính định danh trong thực tế {#bon-thuoc-tinh-trong-thuc-te}

> Phần này không có trong trang gốc. Nó diễn giải kỹ hơn mục [Tên (Names)](#names) bằng ví dụ,
> tập trung vào **API group** — thuộc tính hay bị bỏ qua nhất trong bốn thuộc tính định danh.

### Bốn thuộc tính = địa chỉ đầy đủ của một object

Bốn thuộc tính đó không phải khái niệm trừu tượng — chúng chính là **đường dẫn URL** mà
`kubectl` gọi tới API server:

```
/apis/apps/v1/namespaces/production/deployments/web
      ^^^^ ^^            ^^^^^^^^^^ ^^^^^^^^^^^ ^^^
      group version      namespace  resource    name
```

Bỏ `version` ra, phần còn lại (`apps` + `deployments` + `production` + `web`) là định danh
duy nhất.

Ẩn dụ hành chính cho dễ nhớ:

| Thuộc tính | Tương đương | Ví dụ |
| --- | --- | --- |
| API group | Bộ/Sở quản lý loại giấy tờ | Sở Giao thông |
| Resource type | Loại giấy tờ | Giấy phép lái xe |
| Namespace | Chi nhánh lưu hồ sơ | Chi nhánh Quận 1 |
| Name | Số hồ sơ | `HS-0042` |

Sở Giao thông và Sở Y tế đều có thể có hồ sơ số `HS-0042` mà không đụng nhau, vì khác Bộ/Sở
và khác loại giấy tờ.

#### Ví dụ thực tế bạn gặp hằng ngày

Một app tên `myapp` triển khai bằng Helm sẽ tạo ra một loạt object **trùng tên nhau y hệt**
trong cùng namespace, và không hề xung đột:

```yaml
apiVersion: apps/v1              # group = apps
kind: Deployment
metadata: { name: myapp, namespace: production }
---
apiVersion: v1                   # group = "" (core)
kind: Service
metadata: { name: myapp, namespace: production }
---
apiVersion: networking.k8s.io/v1 # group = networking.k8s.io
kind: Ingress
metadata: { name: myapp, namespace: production }
---
apiVersion: v1                   # group = "" (core)
kind: ConfigMap
metadata: { name: myapp, namespace: production }
```

Bốn object, cùng tên `myapp`, cùng namespace `production`. Hợp lệ vì `(group, resource)` khác
nhau. Đây chính là lý do mọi Helm chart đều đặt tên tất cả object bằng cùng một
`{{ .Release.Name }}` — rất tiện, và Kubernetes cho phép.

Ngược lại, cái này **sai**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp, namespace: production }
---
apiVersion: apps/v1beta1   # đổi version không cứu được
kind: Deployment
metadata: { name: myapp, namespace: production }
```

Vì version **không** nằm trong định danh. Bạn `apply` cái thứ hai thì nó chỉ ghi đè cái thứ
nhất, không tạo thêm object mới.

### API group được dùng thực tế ở đâu khi triển khai dự án

Không phải kiến thức lý thuyết — API group xuất hiện ở những chỗ bạn phải chạm vào trong mọi
dự án thật.

#### 1. Dòng `apiVersion` trong mọi manifest

`apiVersion` chính là `<group>/<version>`. Trường hợp không có dấu `/` là **core group**
(group rỗng, `""`) — nhóm cổ nhất, có từ trước khi Kubernetes tách group.

```yaml
apiVersion: v1                            # core group: Pod, Service, ConfigMap, Secret, Node, PVC
apiVersion: apps/v1                       # Deployment, StatefulSet, DaemonSet, ReplicaSet
apiVersion: batch/v1                      # Job, CronJob
apiVersion: networking.k8s.io/v1          # Ingress, NetworkPolicy
apiVersion: rbac.authorization.k8s.io/v1  # Role, RoleBinding, ClusterRole
apiVersion: storage.k8s.io/v1             # StorageClass, CSIDriver
```

Câu hỏi "tại sao Pod là `v1` mà Deployment là `apps/v1`?" chỉ có một câu trả lời: Pod ở core
group, Deployment ở group `apps`. Không có logic nào sâu xa hơn, chỉ là lịch sử.

#### 2. RBAC — đây là chỗ API group quan trọng nhất

Đây là nơi bạn **bắt buộc** phải viết đúng group, không đoán được. Ví dụ Role cho pipeline
CI/CD chỉ được deploy, không được đọc Secret:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]                  # Deployment nằm ở group apps
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "patch", "update"]

  - apiGroups: [""]                      # <-- chuỗi RỖNG, không phải "core", không phải "v1"
    resources: ["pods", "pods/log", "services", "configmaps"]
    verbs: ["get", "list"]

  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "update"]
```

Lỗi kinh điển khi mới làm RBAC: viết `apiGroups: ["core"]` hoặc `apiGroups: ["v1"]` cho Pod.
Cả hai đều **không báo lỗi khi apply** — API server nhận, rule tồn tại, nhưng nó khớp với một
group không tồn tại nên quyền không bao giờ có hiệu lực. Bạn sẽ ngồi debug `Forbidden` mà
nhìn Role thấy "đã cấp rồi mà". Core group phải viết là `""`.

Lỗi thứ hai: cấp `apiGroups: ["apps"]` kèm `resources: ["pods"]` — sai, vì Pod không ở group
`apps`. Rule vẫn apply thành công và vẫn vô dụng.

#### 3. Gọi tên tài nguyên khi có xung đột giữa các operator

Khi cluster mới tinh, `kubectl get pods` không nhập nhằng. Nhưng dự án thật cài thêm operator,
mỗi operator mang CRD với group riêng, và tên ngắn có thể đụng nhau.

Ví dụ có thật: cài đồng thời Istio và Gateway API thì có hai resource cùng tên `gateways`:

- `gateways.networking.istio.io`
- `gateways.gateway.networking.k8s.io`

Lúc đó `kubectl get gateway` trở nên mơ hồ, kubectl chọn một cái theo thứ tự ưu tiên chứ không
hỏi bạn. Cách viết chắc chắn là gọi kèm group:

```bash
kubectl get gateways.networking.istio.io -n production
kubectl get gateways.gateway.networking.k8s.io -n production
```

Cú pháp `<resource>.<group>` này dùng được ở mọi chỗ: `kubectl get deployments.apps`,
`kubectl delete ingresses.networking.k8s.io myapp`.

Ngay trong Kubernetes gốc cũng có ví dụ: `events` tồn tại ở **cả** core group (`events`) lẫn
group `events.k8s.io` (`events.events.k8s.io`) — hai cách biểu diễn của cùng dữ liệu bên dưới.

#### 4. Khi bạn tự viết CRD — group là "họ" của dự án bạn

Viết CRD cho hệ thống nội bộ thì bạn phải tự chọn group, và quy ước là dùng **domain công ty
bạn sở hữu**:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.platform.example.com   # <plural>.<group>
spec:
  group: platform.example.com            # <-- group của bạn
  names:
    plural: databases
    singular: database
    kind: Database
```

Sau đó object của bạn khai báo `apiVersion: platform.example.com/v1alpha1`. Group đóng vai trò
như "namespace của tên loại tài nguyên": bạn đặt tên `Database` thoải mái, không sợ đụng
`Database` của một vendor khác, vì group khác nhau.

Đây cũng là lý do đừng bao giờ đặt CRD vào group `*.k8s.io` — group đó thuộc về upstream
Kubernetes, nâng cấp cluster có thể đụng độ.

#### 5. Nâng cấp cluster — rà `apiVersion` chính là rà group/version

Khi lên version Kubernetes mới, thứ bị gỡ bỏ luôn được thông báo theo dạng
`<group>/<version>`, ví dụ:

- `policy/v1beta1` PodSecurityPolicy → gỡ hẳn từ 1.25
- `batch/v1beta1` CronJob → chuyển sang `batch/v1`
- `networking.k8s.io/v1beta1` Ingress → chuyển sang `networking.k8s.io/v1`

Công việc chuẩn bị nâng cấp trong dự án thật là quét toàn bộ manifest/Helm chart tìm những
`apiVersion` sắp bị gỡ. Nếu bạn không hiểu `apiVersion` = group + version thì không đọc nổi
release note.

#### 6. Aggregated API — group không do API server phục vụ

`kubectl top pods` gọi vào group `metrics.k8s.io`, nhưng group này **không** do kube-apiserver
phục vụ; nó được ủy quyền cho metrics-server qua object `APIService`. Vì vậy khi chưa cài
metrics-server, lỗi bạn nhận được là:

```
error: Metrics API not available
```

Đó không phải lỗi RBAC hay lỗi mạng — đó là "group `metrics.k8s.io` đã đăng ký nhưng không có
ai đứng sau nó". Hiểu group giúp chẩn đoán đúng chỗ ngay.

### Lệnh để tự kiểm chứng trên cluster

```bash
# Liệt kê mọi resource kèm group/version, tên ngắn, có namespace hay không
kubectl api-resources

# Chỉ xem những gì thuộc group apps
kubectl api-resources --api-group=apps

# Xem tất cả group/version cluster đang phục vụ
kubectl api-versions

# Xem thẳng cây API dạng REST
kubectl get --raw /apis | head -c 500
```

Cột `APIVERSION` trong `kubectl api-resources` chính là thứ bạn chép vào `apiVersion` của
manifest, và phần trước dấu `/` chính là thứ bạn chép vào `apiGroups` của Role. Với dòng nào
chỉ hiện `v1` (Pod, Service, ConfigMap...), `apiGroups` phải là `""`.

### Một câu tóm gọn

> API group là **cách Kubernetes chia API thành các họ độc lập** để tên tài nguyên không đụng
> nhau và để từng họ tiến hóa version riêng. Trong dự án thật bạn gặp nó ở `apiVersion` của
> manifest, ở `apiGroups` của RBAC, ở việc gọi đúng CRD khi nhiều operator cùng tồn tại, ở
> việc chọn group cho CRD của mình, và ở mọi thông báo deprecation khi nâng cấp cluster.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Trong cùng một namespace, có thể vừa có Pod tên `web` vừa có Deployment tên `web` không?
   Còn hai Pod cùng tên `web` ở hai namespace khác nhau?
2. Kể bốn thuộc tính định danh duy nhất một object. Phiên bản API có nằm trong đó không, và
   hệ quả là gì?
3. Bạn xóa Pod `web` rồi tạo lại một Pod cũng tên `web`. Object mới có cùng UID với object cũ
   không? Vì sao Kubernetes thiết kế như vậy?
4. Ở Lab 1a phần B8, bạn so sánh giá trị nào để chứng minh ServiceAccount `default` được
   controller **tạo mới** chứ không phải chưa từng bị xóa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Được cả hai.** Name chỉ cần duy nhất trong phạm vi *(API group, resource type, namespace)*.
   Pod và Deployment là hai resource type khác nhau nên không đụng nhau; hai namespace khác
   nhau cũng là hai phạm vi khác nhau.
2. API group, resource type, namespace, name. **Phiên bản API không nằm trong định danh** —
   các version chỉ là cách biểu diễn khác của cùng một dữ liệu. Hệ quả: không thể tạo hai
   object trùng tên cùng resource type trong cùng namespace bằng cách dùng `v1` và `v1beta1`.
3. **Không** — UID mới. UID sinh ra để phân biệt các lần xuất hiện trong lịch sử của những
   thực thể trông giống nhau, nên hệ thống luôn nói được "đây là object khác, không phải cái
   cũ hồi sinh".
4. So sánh **UID** trước và sau. Tên vẫn là `default` nên tên không chứng minh được gì; UID
   khác mới chứng minh đây là object mới do controller tạo.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
