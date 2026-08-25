# Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/>
>
> Định nghĩa giới hạn tài nguyên bộ nhớ mặc định cho một namespace, để mọi Pod mới
> trong namespace đó đều được cấu hình giới hạn tài nguyên bộ nhớ.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace)
— **trang con của bài 2/7**, mục [Quản lý tài nguyên Memory, CPU và API](228-manage-resources-tasks-vi.md);
nó không có dòng riêng trong lộ trình. Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối [mục giai đoạn
25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace).

Trang này là bản memory của [230](230-cpu-default-namespace-vi.md): cặp thứ hai trong sáu trang
con — **LimitRange đặt `default`/`defaultRequest`**, **điền vào** Pod không khai gì, khác hẳn cặp
`min`/`max` ở [231](231-memory-constraint-namespace-vi.md) vốn **từ chối** Pod khai ngoài khoảng.
Bản memory có thêm **một ghi chú mà bản CPU không có**: LimitRange không kiểm tra tính nhất quán
giữa `default` và `requests` của client — đúng cái bẫy bạn đã dựng ở
[Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) phần B2.6. Việc LimitRange chèn giá trị mặc
định vào Pod trần thì phần B2.3 của lab đó đã làm rồi.

**Phải hiểu ở lần đọc này:**

- Pod **không khai gì** trong namespace có `default: 512Mi` và `defaultRequest: 256Mi` nhận đúng
  hai giá trị đó: `requests.memory: 256Mi`, `limits.memory: 512Mi`.
- Mục *Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request?*: request
  được đặt **bằng chính limit** (`1Gi`), và bài nhấn mạnh container **không** nhận `defaultRequest`
  256Mi. Câu mở đầu trang đã báo trước — memory request mặc định chỉ được gán **trong một số điều
  kiện nhất định**.
- Mục *Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit?*: request
  **giữ nguyên** `128Mi` như manifest, còn limit lấy `default` của namespace là `512Mi`.
- Ghi chú riêng của trang này: **`LimitRange` không kiểm tra tính nhất quán** của các giá trị mặc
  định nó áp. `default` (limit) có thể **nhỏ hơn** `requests` mà client gửi lên, và khi đó Pod
  **không lập lịch được** — hỏng ở khâu scheduling, không phải bị từ chối lúc tạo.
- Mục *Động lực cho memory limit và request mặc định* liệt kê **ba** ràng buộc của một memory
  ResourceQuota, nhiều hơn bản CPU một ràng buộc: mọi Pod **và từng container** phải có memory
  limit; tổng lượng bộ nhớ **được đặt trước** của mọi Pod trong namespace không vượt trần; và
  tổng lượng bộ nhớ **thực sự được sử dụng** cũng không vượt trần.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Câu trong ngoặc ở ràng buộc thứ nhất — Kubernetes suy ra memory limit **ở cấp Pod** bằng cách cộng limit của các container | tài nguyên khai ở cấp Pod là một bài riêng, không phải nội dung của LimitRange | bài [265](265-assign-pod-level-resources-vi.md), dòng Thực hành của [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |
| Chi tiết cách đặt và đọc một memory ResourceQuota (hai ràng buộc về tổng lượng đặt trước và tổng lượng dùng thật) | trang này chỉ nêu để giải thích **vì sao cần giá trị mặc định** | bài [233](233-quota-memory-cpu-namespace-vi.md), trang con kế tiếp của bài 2/7 |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho quản trị viên cluster* và *Dành cho nhà phát triển ứng dụng* | là con trỏ, không có nội dung mới | các trang còn lại của bài 2/7 đọc ngay sau đây; nhánh nhà phát triển đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này hướng dẫn cách cấu hình memory request và limit mặc định cho một namespace.

Một cluster Kubernetes có thể được chia thành nhiều namespace. Khi bạn đã có một namespace
với memory
[limit](110-manage-resources-containers-vi.md#requests-and-limits)
mặc định, và sau đó bạn thử tạo một Pod có container không chỉ định memory limit của riêng nó,
thì control plane sẽ gán memory limit mặc định cho container đó.

Kubernetes gán memory request mặc định trong một số điều kiện nhất định sẽ được giải thích
ở phần sau của bài này.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn phải có quyền tạo namespace trong cluster của mình.

Mỗi node trong cluster của bạn phải có ít nhất 2 GiB bộ nhớ.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace default-mem-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest cho một LimitRange ví dụ. Manifest này chỉ định một memory request
mặc định và một memory limit mặc định.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

Tạo LimitRange trong namespace default-mem-example:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults.yaml --namespace=default-mem-example
```

Bây giờ nếu bạn tạo một Pod trong namespace default-mem-example, và bất kỳ container nào
trong Pod đó không chỉ định giá trị memory request và memory limit của riêng nó,
thì control plane sẽ áp dụng các giá trị mặc định: memory request là 256MiB và
memory limit là 512MiB.

Dưới đây là manifest ví dụ cho một Pod có một container. Container này không chỉ định
memory request và limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo
spec:
  containers:
  - name: default-mem-demo-ctr
    image: nginx
```

Tạo Pod.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod.yaml --namespace=default-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod default-mem-demo --output=yaml --namespace=default-mem-example
```

Output cho thấy container của Pod có memory request là 256 MiB và memory limit là 512 MiB.
Đây là các giá trị mặc định do LimitRange chỉ định.

```shell
containers:
- image: nginx
  imagePullPolicy: Always
  name: default-mem-demo-ctr
  resources:
    limits:
      memory: 512Mi
    requests:
      memory: 256Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod default-mem-demo --namespace=default-mem-example
```

## Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request? (What if you specify a container's limit, but not its request?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory limit,
nhưng không chỉ định request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-2
spec:
  containers:
  - name: default-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1Gi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-2.yaml --namespace=default-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod default-mem-demo-2 --output=yaml --namespace=default-mem-example
```

Output cho thấy memory request của container được đặt bằng với memory limit của nó.
Lưu ý rằng container không được gán giá trị memory request mặc định 256Mi.

```
resources:
  limits:
    memory: 1Gi
  requests:
    memory: 1Gi
```

## Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit? (What if you specify a container's request, but not its limit?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory request,
nhưng không chỉ định limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-3
spec:
  containers:
  - name: default-mem-demo-3-ctr
    image: nginx
    resources:
      requests:
        memory: "128Mi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-3.yaml --namespace=default-mem-example
```

Xem specification của Pod:

```shell
kubectl get pod default-mem-demo-3 --output=yaml --namespace=default-mem-example
```

Output cho thấy memory request của container được đặt theo giá trị chỉ định trong manifest
của container. Container bị giới hạn sử dụng không quá 512MiB bộ nhớ, khớp với memory limit
mặc định của namespace này.

```
resources:
  limits:
    memory: 512Mi
  requests:
    memory: 128Mi
```

> **Ghi chú:**
>
> Một `LimitRange` **không** kiểm tra tính nhất quán của các giá trị mặc định mà nó áp dụng.
> Điều này có nghĩa là giá trị mặc định cho _limit_ do `LimitRange` đặt có thể nhỏ hơn giá trị
> _request_ được chỉ định cho container trong spec mà client gửi tới API server. Nếu điều đó
> xảy ra, Pod cuối cùng sẽ không thể được lập lịch (schedule).
> Xem [Ràng buộc đối với resource limit và request](133-limit-range-vi.md#constraints-on-resource-limits-and-requests)
> để biết thêm chi tiết.

## Động lực cho memory limit và request mặc định (Motivation for default memory limits and requests)

Nếu namespace của bạn đã cấu hình hạn ngạch tài nguyên (resource quota) cho bộ nhớ,
thì việc có sẵn một giá trị mặc định cho memory limit là rất hữu ích.
Dưới đây là ba trong số các ràng buộc mà một hạn ngạch tài nguyên áp đặt lên một namespace:

* Với mọi Pod chạy trong namespace, Pod và mỗi container của nó phải có memory limit.
  (Nếu bạn chỉ định memory limit cho mọi container trong một Pod, Kubernetes có thể suy ra
  memory limit ở cấp Pod bằng cách cộng các limit của các container lại).
* Memory limit áp dụng một lượng tài nguyên được đặt trước (resource reservation) trên node
  mà Pod đó được lập lịch. Tổng lượng bộ nhớ được đặt trước cho tất cả các Pod trong namespace
  không được vượt quá một giới hạn đã chỉ định.
* Tổng lượng bộ nhớ thực sự được sử dụng bởi tất cả các Pod trong namespace cũng không được
  vượt quá một giới hạn đã chỉ định.

Khi bạn thêm một LimitRange:

Nếu bất kỳ Pod nào trong namespace đó chứa một container không chỉ định memory limit của
riêng nó, control plane sẽ áp dụng memory limit mặc định cho container đó, và Pod có thể
được phép chạy trong một namespace đang bị giới hạn bởi một memory ResourceQuota.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace default-mem-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình CPU request và limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình hạn ngạch bộ nhớ và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình hạn ngạch Pod cho một Namespace](234-quota-pod-namespace-vi.md)

* [Cấu hình hạn ngạch cho các API Object](252-quota-api-object-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên bộ nhớ cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và bộ nhớ ở cấp Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. Trong namespace có LimitRange `mem-limit-range`, Pod `default-mem-demo` không khai gì cả. Hai
   giá trị cuối cùng của container là bao nhiêu, và mỗi giá trị lấy từ field nào?
2. **Câu bẫy.** Pod `default-mem-demo-2` khai `limits.memory: "1Gi"` và bỏ trống request, trong
   khi namespace có `defaultRequest: 256Mi`. Request cuối cùng là bao nhiêu? Đặt cạnh Pod
   `default-mem-demo-3` (khai request `128Mi`, bỏ trống limit), rút ra quy tắc nào?
3. Bài có một ghi chú nói `LimitRange` **không** kiểm tra tính nhất quán của giá trị mặc định nó
   áp. Dựng một tình huống cụ thể khiến điều đó gây hỏng, và cho biết Pod hỏng ở **bước nào** —
   bị từ chối lúc tạo, hay tạo được rồi mà không chạy?
4. Bạn định đặt một memory ResourceQuota cho namespace của một nhóm trên cluster lab (mỗi worker
   6 GB RAM). Mục *Động lực* liệt kê ba ràng buộc mà quota áp đặt — kể lại, và nói rõ ràng buộc
   nào làm cho việc kèm một LimitRange mặc định trở thành bắt buộc trên thực tế.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Container nhận **`requests.memory: 256Mi`** — từ **`defaultRequest`** — và
   **`limits.memory: 512Mi`** — từ **`default`**. Đây là trường hợp duy nhất trong bài mà cả hai
   giá trị đều do LimitRange cấp.
2. Request cuối cùng là **`1Gi`, bằng đúng limit** — bài viết rõ: memory request của container
   được đặt bằng với memory limit của nó, và container **không được gán giá trị memory request mặc
   định 256Mi**. Đặt cạnh Pod thứ ba (giữ `requests.memory: 128Mi`, nhận `limits.memory: 512Mi` từ
   `default`), quy tắc là: **giá trị bạn tự khai luôn thắng**, còn phần thiếu được điền từ **hai
   nguồn khác nhau** — thiếu limit thì lấy `default` của namespace, nhưng thiếu request khi đã có
   limit thì lấy **chính limit đó**, không lấy `defaultRequest`.
3. Tình huống: namespace đặt `default` (limit) là `512Mi`, còn client gửi lên một container khai
   `requests.memory: "1Gi"` mà **không** khai limit. LimitRange điền `limits.memory: 512Mi` và
   **không kiểm tra** rằng nó nhỏ hơn request `1Gi`. Kết quả: Pod **được tạo** — nó không vi phạm
   ràng buộc nào để bị từ chối — nhưng **không thể được lập lịch**. Nghĩa là hỏng ở **khâu
   scheduling**, và triệu chứng là Pod nằm chờ chứ không phải một thông báo `Forbidden`.
4. Ba ràng buộc: **(a)** mọi Pod trong namespace, và **từng container** của nó, phải có memory
   limit; **(b)** tổng lượng bộ nhớ **được đặt trước** cho tất cả Pod trong namespace không được
   vượt quá một giới hạn đã chỉ định; **(c)** tổng lượng bộ nhớ **thực sự được sử dụng** cũng
   không được vượt quá một giới hạn đã chỉ định. Ràng buộc **(a)** là cái buộc phải kèm LimitRange:
   chỉ cần một container trong namespace quên khai memory limit là Pod đó không được phép chạy.
   Có LimitRange mặc định thì control plane **áp limit mặc định cho container thiếu**, và Pod đi
   qua được quota mà người dùng không phải sửa manifest.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
