# Các đối tượng trong Kubernetes (Objects In Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/>
>
> Đối tượng Kubernetes (Kubernetes objects) là những thực thể bền vững (persistent entities) trong hệ thống Kubernetes.
> Kubernetes dùng các thực thể này để biểu diễn trạng thái của cluster.
> Tìm hiểu về mô hình đối tượng Kubernetes và cách làm việc với các đối tượng này.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](00-ALO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 4/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B4 và B5.3.

**Phải hiểu ở lần đọc này:**

- Object là một **bản ghi ý định**: tạo object tức là khai báo bạn muốn cluster trông thế nào,
  rồi hệ thống liên tục làm việc để giữ đúng như vậy.
- `spec` là trạng thái **mong muốn** do bạn đặt; `status` là trạng thái **thực tế** do hệ
  thống quan sát rồi ghi. Đây là ý quan trọng nhất của bài.
- Bốn trường bắt buộc trong manifest: `apiVersion`, `kind`, `metadata`, `spec`.
- Server-side field validation và ba mức `ignore` / `warn` / `strict`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung bên trong manifest Deployment ví dụ (`selector`, `matchLabels`, `template`, `replicas`) | chưa học label selector và Deployment | nhóm 1b và giai đoạn 4 |
| Liên kết tới API Reference, API Conventions | tài liệu tra cứu, không đọc tuần tự | khi cần tra field |

Với manifest ví dụ, lần này chỉ cần nhìn ra **hình dạng bốn trường bắt buộc**, không cần hiểu
nội dung bên trong `spec`.

---

Trang này giải thích cách các đối tượng Kubernetes được biểu diễn trong Kubernetes API, và cách bạn
thể hiện chúng ở định dạng `.yaml`.

## Hiểu về các đối tượng Kubernetes (Understanding Kubernetes objects) {#kubernetes-objects}

*Đối tượng Kubernetes (Kubernetes objects)* là những thực thể bền vững trong hệ thống Kubernetes. Kubernetes dùng các
thực thể này để biểu diễn trạng thái của cluster. Cụ thể, chúng có thể mô tả:

* Những ứng dụng đóng gói trong container (containerized applications) nào đang chạy (và trên node nào)
* Các tài nguyên khả dụng cho những ứng dụng đó
* Các chính sách (policy) về cách những ứng dụng đó hoạt động, chẳng hạn chính sách khởi động lại (restart), nâng cấp, và khả năng chịu lỗi (fault-tolerance)

Một đối tượng Kubernetes là một "bản ghi ý định" (record of intent) — một khi bạn tạo đối tượng, hệ thống Kubernetes
sẽ liên tục làm việc để bảo đảm đối tượng đó tồn tại. Bằng việc tạo một đối tượng, thực chất bạn
đang nói cho hệ thống Kubernetes biết bạn muốn workload của cluster trông như thế nào; đây chính là
*trạng thái mong muốn (desired state)* của cluster.

Để làm việc với các đối tượng Kubernetes — dù là tạo, sửa đổi hay xóa chúng — bạn cần dùng
[Kubernetes API](21-kubernetes-api-vi.md). Ví dụ, khi bạn dùng giao diện
dòng lệnh `kubectl`, CLI sẽ thực hiện các lời gọi Kubernetes API cần thiết thay cho bạn. Bạn cũng có thể dùng
Kubernetes API trực tiếp trong chương trình của riêng mình thông qua một trong các
[thư viện client (Client Libraries)](https://kubernetes.io/docs/reference/using-api/client-libraries/).

### Spec và status của đối tượng (Object spec and status) {#object-spec-and-status}

Hầu như mọi đối tượng Kubernetes đều bao gồm hai trường đối tượng lồng nhau chi phối
cấu hình của đối tượng: *`spec`* và *`status`* của đối tượng.
Với những đối tượng có `spec`, bạn phải thiết lập trường này khi tạo đối tượng,
cung cấp mô tả về các đặc tính bạn muốn tài nguyên (resource) có:
_trạng thái mong muốn (desired state)_ của nó.

`status` mô tả _trạng thái hiện tại (current state)_ của đối tượng, do hệ thống Kubernetes
và các thành phần của nó cung cấp và cập nhật. Control plane của Kubernetes liên tục
và chủ động quản lý trạng thái thực tế của mọi đối tượng sao cho khớp với trạng thái mong muốn
mà bạn đã cung cấp.

Ví dụ: trong Kubernetes, Deployment là một đối tượng có thể đại diện cho một
ứng dụng chạy trên cluster của bạn. Khi tạo Deployment, bạn có thể
đặt `spec` của Deployment để chỉ định rằng bạn muốn ba bản sao (replica) của
ứng dụng được chạy. Hệ thống Kubernetes đọc spec của Deployment và khởi chạy ba
thực thể (instance) của ứng dụng bạn mong muốn — cập nhật status sao cho khớp với
spec của bạn. Nếu bất kỳ instance nào trong số đó gặp sự cố
(một thay đổi về status), hệ thống Kubernetes phản ứng với sự khác biệt
giữa spec và status bằng cách thực hiện chỉnh sửa — trong trường hợp này là
khởi chạy một instance thay thế.

Để biết thêm thông tin về spec, status và metadata của đối tượng, xem
[Kubernetes API Conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md).

### Mô tả một đối tượng Kubernetes (Describing a Kubernetes object)

Khi tạo một đối tượng trong Kubernetes, bạn phải cung cấp spec của đối tượng mô tả
trạng thái mong muốn của nó, cùng một số thông tin cơ bản về đối tượng (chẳng hạn tên). Khi bạn dùng
Kubernetes API để tạo đối tượng (trực tiếp hoặc thông qua `kubectl`), yêu cầu API đó phải
chứa các thông tin này dưới dạng JSON trong phần thân (body) của request.
Thông thường nhất, bạn cung cấp thông tin cho `kubectl` trong một file được gọi là _manifest_.
Theo quy ước, manifest ở dạng YAML (bạn cũng có thể dùng định dạng JSON).
Các công cụ như `kubectl` chuyển đổi thông tin từ manifest sang JSON hoặc một định dạng
tuần tự hóa (serialization) được hỗ trợ khác khi thực hiện yêu cầu API qua HTTP.

Dưới đây là một manifest ví dụ cho thấy các trường bắt buộc và spec của đối tượng cho một
Deployment trong Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # yêu cầu deployment chạy 2 pod khớp với template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Một cách để tạo Deployment bằng file manifest như trên là dùng lệnh
[`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply)
trong giao diện dòng lệnh `kubectl`, truyền file `.yaml` làm đối số. Ví dụ:

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

Kết quả xuất ra tương tự như sau:

```
deployment.apps/nginx-deployment created
```

### Các trường bắt buộc (Required fields)

Trong manifest (file YAML hoặc JSON) cho đối tượng Kubernetes bạn muốn tạo, bạn cần đặt giá trị cho
các trường sau:

* `apiVersion` - Phiên bản Kubernetes API mà bạn đang dùng để tạo đối tượng này
* `kind` - Loại đối tượng bạn muốn tạo
* `metadata` - Dữ liệu giúp định danh duy nhất đối tượng, bao gồm chuỗi `name`, `UID`, và `namespace` (tùy chọn)
* `spec` - Trạng thái bạn mong muốn cho đối tượng

Định dạng chính xác của `spec` là khác nhau đối với từng đối tượng Kubernetes, và chứa
các trường lồng nhau đặc thù cho đối tượng đó. [Tài liệu tham khảo Kubernetes API (Kubernetes API Reference)](https://kubernetes.io/docs/reference/kubernetes-api/)
có thể giúp bạn tìm định dạng spec cho tất cả các đối tượng bạn có thể tạo bằng Kubernetes.

Ví dụ, xem [trường `spec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)
trong tài liệu tham khảo API của Pod.
Với mỗi Pod, trường `.spec` chỉ định pod đó và trạng thái mong muốn của nó (chẳng hạn tên image container cho
từng container bên trong pod đó).
Một ví dụ khác về đặc tả đối tượng là
[trường `spec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/#StatefulSetSpec)
của StatefulSet API. Với StatefulSet, trường `.spec` chỉ định StatefulSet đó và
trạng thái mong muốn của nó.
Bên trong `.spec` của một StatefulSet là một [template](https://kubernetes.io/docs/concepts/workloads/pods#pod-templates)
cho các đối tượng Pod. Template đó mô tả các Pod mà controller của StatefulSet sẽ tạo ra nhằm
thỏa mãn đặc tả của StatefulSet.
Các loại đối tượng khác nhau cũng có thể có `.status` khác nhau; một lần nữa, các trang tham khảo API
mô tả chi tiết cấu trúc của trường `.status` đó, cùng nội dung của nó cho từng loại đối tượng khác nhau.

Xem [Kubernetes Configuration Best Practices](https://kubernetes.io/blog/2025/11/25/configuration-good-practices/)
để có thêm thông tin về cách viết file cấu hình YAML.

## Kiểm tra hợp lệ trường phía server (Server side field validation)

Bắt đầu từ Kubernetes v1.25, API server cung cấp tính năng
[kiểm tra hợp lệ trường (field validation)](https://kubernetes.io/docs/reference/using-api/api-concepts/#field-validation)
phía server, giúp phát hiện các trường không được nhận diện hoặc bị trùng lặp trong một đối tượng.
Nó cung cấp toàn bộ chức năng của `kubectl --validate` ở phía server.

Công cụ `kubectl` dùng cờ (flag) `--validate` để đặt mức kiểm tra hợp lệ trường. Cờ này chấp nhận các
giá trị `ignore`, `warn`, và `strict`, đồng thời cũng chấp nhận giá trị `true` (tương đương `strict`)
và `false` (tương đương `ignore`). Thiết lập kiểm tra hợp lệ mặc định của `kubectl` là `--validate=true`.

`Strict`
: Kiểm tra hợp lệ trường nghiêm ngặt, báo lỗi khi việc kiểm tra thất bại

`Warn`
: Việc kiểm tra hợp lệ trường vẫn được thực hiện, nhưng lỗi được đưa ra dưới dạng cảnh báo thay vì làm request thất bại

`Ignore`
: Không thực hiện kiểm tra hợp lệ trường phía server

Khi `kubectl` không thể kết nối tới một API server hỗ trợ kiểm tra hợp lệ trường, nó sẽ quay về (fall back)
dùng cơ chế kiểm tra hợp lệ phía client. Kubernetes 1.27 và các phiên bản sau luôn cung cấp
kiểm tra hợp lệ trường; các bản phát hành Kubernetes cũ hơn có thể không. Nếu cluster của bạn cũ hơn v1.27,
hãy xem tài liệu cho phiên bản Kubernetes của bạn.

## Tiếp theo (What's next)

Nếu bạn mới làm quen với Kubernetes, hãy đọc thêm về các nội dung sau:

* [Pods](46-pods-vi.md) — những đối tượng Kubernetes cơ bản quan trọng nhất.
* Các đối tượng [Deployment](63-deployment-vi.md).
* [Controllers](25-controllers-vi.md) trong Kubernetes.
* [kubectl](https://kubernetes.io/docs/reference/kubectl/) và [các lệnh kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands).

[Quản lý đối tượng Kubernetes (Kubernetes Object Management)](27-object-management-vi.md)
giải thích cách dùng `kubectl` để quản lý các đối tượng.
Bạn có thể cần [cài đặt kubectl](185-tools-vi.md#kubectl) nếu chưa có sẵn công cụ này.

Để tìm hiểu về Kubernetes API nói chung, hãy xem:

* [Tổng quan Kubernetes API (Kubernetes API overview)](https://kubernetes.io/docs/reference/using-api/)

Để tìm hiểu sâu hơn về các đối tượng trong Kubernetes, hãy đọc các trang khác trong phần này:

* [Quản lý đối tượng Kubernetes (Kubernetes Object Management)](27-object-management-vi.md)
* [Tên và ID của đối tượng (Object Names and IDs)](./17-names-vi.md)
* [Labels và Selectors (Labels and Selectors)](18-labels-vi.md)
* [Namespaces](19-namespaces-vi.md)
* [Annotations](20-annotations-vi.md)
* [Field Selectors](28-field-selectors-vi.md)
* [Finalizers](29-finalizers-vi.md)
* [Owners và Dependents (Owners and Dependents)](30-owners-dependents-vi.md)
* [Các label khuyến nghị (Recommended Labels)](31-common-labels-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Trong một Node object, `spec.podCIDR` và `status.capacity.cpu` — trường nào do bạn hoặc
   cluster đặt, trường nào do hệ thống quan sát rồi ghi vào?
2. Vì sao client không nên tự ý ghi vào `status`?
3. Bạn `kubectl apply` một YAML có tên field sai chính tả. Với `--validate=strict`, chuyện gì
   xảy ra? Object có được tạo không?
4. Kể bốn trường bắt buộc của một manifest và nói mỗi trường trả lời câu hỏi gì.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `spec.podCIDR` nằm trong **`spec`** — trạng thái mong muốn, do cluster gán cho node.
   `status.capacity.cpu` nằm trong **`status`** — trạng thái thực tế, do kubelet quan sát phần
   cứng rồi báo lên. Quy tắc nhận biết nhanh: nhìn tên nhánh chứa nó.
2. Vì `status` là nơi **hệ thống công bố những gì nó quan sát được**. Client ghi vào đó là nói
   dối về thực tế; control plane vẫn tiếp tục cập nhật đè lên theo quan sát thật, nên giá trị
   bịa vừa vô nghĩa vừa làm các controller đang đọc `status` ra quyết định sai.
3. API server **từ chối request** và báo có field không nhận diện được; **object không được
   tạo**. Với `--validate=warn` thì chỉ cảnh báo và vẫn tạo; với `ignore` thì không kiểm tra
   phía server.
4. `apiVersion` — dùng phiên bản API nào? `kind` — loại object gì? `metadata` — object này là
   ai (name, UID, namespace)? `spec` — bạn muốn nó ở trạng thái nào?

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
