# Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/>
>
> Định nghĩa một khoảng giá trị CPU resource limit hợp lệ cho một namespace, để mọi Pod mới
> trong namespace đó nằm trong khoảng mà bạn cấu hình.

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

Sáu trang con của bài 2/7 chia làm ba cặp, và **ranh giới giữa chúng là chỗ dễ nhầm nhất của cả
giai đoạn 25**. Trang này thuộc cặp thứ nhất: **LimitRange đặt `min`/`max`** — **từ chối** Pod
khai ngoài khoảng. Cặp thứ hai ([230](230-cpu-default-namespace-vi.md),
[232](232-memory-default-namespace-vi.md)) là LimitRange đặt `default`/`defaultRequest` — **điền
vào** Pod không khai gì. Cặp thứ ba ([233](233-quota-memory-cpu-namespace-vi.md),
[234](234-quota-pod-namespace-vi.md)) là **ResourceQuota** — trần của **cả namespace**.

Lý thuyết bạn đã đọc ở bài [133](133-limit-range-vi.md) và [134](134-resource-quotas-vi.md) (giai
đoạn 7b), và [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) đã thực hành đúng phần
`min`/`max` từ chối ngay lúc tạo (phần B2.4, B2.5) cùng việc sửa LimitRange không đụng Pod đang
chạy (phần B2.7). Cái giai đoạn 25 thêm vào ở trang này là **hai chi tiết chưa gặp**: LimitRange
chỉ khai `min`/`max` thì Kubernetes **tự sinh** `default` và `defaultRequest`, và **câu chữ chính
xác của hai thông báo từ chối** — thứ mà Checkpoint giai đoạn 25 bắt bạn đọc đúng.

**Phải hiểu ở lần đọc này:**

- Hậu quả và thời điểm, nêu ngay ở đoạn mở đầu và nói lại ở mục *Việc thực thi các ràng buộc CPU
  tối thiểu và tối đa*: Pod không đáp ứng ràng buộc thì **không tạo được** trong namespace đó; và
  ràng buộc chỉ được thực thi **khi Pod được tạo hoặc cập nhật** — sửa LimitRange **không** ảnh
  hưởng tới Pod đã tạo trước đó.
- Tác dụng phụ ở mục *Tạo một LimitRange và một Pod*: manifest chỉ khai `max: 800m` và
  `min: 200m`, nhưng `kubectl get limitrange --output=yaml` in ra thêm **`default: 800m` và
  `defaultRequest: 800m`** — bài nói thẳng là chúng được tạo tự động.
- Ba việc control plane làm với mỗi Pod mới trong namespace, đúng theo thứ tự bài liệt kê: gán
  request/limit **mặc định** cho container nào không tự khai → xác minh mọi container có request
  **≥ 200 millicpu** → xác minh mọi container có limit **≤ 800 millicpu**.
- Hai thông báo từ chối và cách đọc chúng — chất liệu trực tiếp của Checkpoint giai đoạn 25:
  `maximum cpu usage per Container is 800m, but limit is 1500m` (đụng `max`, đối chiếu với
  **limit**) và `minimum cpu usage per Container is 200m, but request is 100m` (đụng `min`, đối
  chiếu với **request**). Cả hai là `Error from server (Forbidden)`.
- Pod `constraints-cpu-demo-4` không khai gì cả **vẫn được tạo** và nhận `800m/800m`, vì control
  plane áp [CPU request và limit mặc định](230-cpu-default-namespace-vi.md) sinh tự động ở gạch
  đầu dòng thứ hai. Bài còn nhắc điều kiện tiên quyết: mỗi Node phải có ít nhất **1.0 CPU** khả
  dụng, nếu không Pod tạo ra rồi vẫn có thể không chạy được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú về LimitRange cho **huge-pages** và **GPU** (`default` và `defaultRequest` phải bằng nhau) | hai worker VM không cấu hình hugepages và cluster lab không có GPU, nên không có gì để đối chiếu | GPU qua device plugin ở [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài [184](184-device-plugins-vi.md); [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) đã ghi đúng lý do hugepages không đo được |
| Ghi chú về **in-place Pod resize** và việc resize bị từ chối khi vi phạm ràng buộc | resize tại chỗ là một bài riêng, không phải nội dung của LimitRange | bài [289](289-resize-container-resources-vi.md), dòng Thực hành của [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho người quản trị cluster* và *Dành cho lập trình viên ứng dụng* | là con trỏ, không có nội dung mới | năm trang còn lại của bài 2/7 đọc ngay sau đây; nhánh lập trình viên đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này chỉ ra cách đặt giá trị tối thiểu và tối đa cho tài nguyên CPU mà các container và
Pod trong một namespace được sử dụng. Bạn chỉ định các giá trị CPU tối thiểu và tối đa trong
một object
[LimitRange](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/).
Nếu một Pod không đáp ứng các ràng buộc mà LimitRange áp đặt, nó không thể được tạo trong
namespace đó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn phải có quyền tạo namespace trong cluster của mình.

Mỗi node trong cluster của bạn phải có ít nhất 1.0 CPU khả dụng cho các Pod.
Xem [ý nghĩa của CPU](110-manage-resources-containers-vi.md#meaning-of-cpu)
để biết Kubernetes hiểu "1 CPU" nghĩa là gì.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace constraints-cpu-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Đây là manifest cho một LimitRange mẫu:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
```

Tạo LimitRange:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints.yaml --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về LimitRange:

```shell
kubectl get limitrange cpu-min-max-demo-lr --output=yaml --namespace=constraints-cpu-example
```

Output cho thấy các ràng buộc CPU tối thiểu và tối đa đúng như mong đợi. Nhưng hãy để ý rằng
mặc dù bạn không chỉ định các giá trị mặc định trong file cấu hình của LimitRange, chúng vẫn
được tạo tự động.

```yaml
limits:
- default:
    cpu: 800m
  defaultRequest:
    cpu: 800m
  max:
    cpu: 800m
  min:
    cpu: 200m
  type: Container
```

Bây giờ, mỗi khi bạn tạo một Pod trong namespace constraints-cpu-example (hoặc một client
khác của Kubernetes API tạo một Pod tương đương), Kubernetes thực hiện các bước sau:

* Nếu bất kỳ container nào trong Pod đó không chỉ định CPU request và limit của riêng nó,
  control plane sẽ gán CPU request và limit mặc định cho container đó.

* Xác minh rằng mọi container trong Pod đó chỉ định một CPU request lớn hơn hoặc bằng 200
  millicpu.

* Xác minh rằng mọi container trong Pod đó chỉ định một CPU limit nhỏ hơn hoặc bằng 800
  millicpu.

> **Ghi chú:**
>
> Khi tạo một object `LimitRange`, bạn cũng có thể chỉ định giới hạn cho huge-pages hoặc GPU.
> Tuy nhiên, khi cả `default` và `defaultRequest` đều được chỉ định trên các tài nguyên này,
> hai giá trị đó phải bằng nhau.

Đây là manifest cho một Pod có một container. Manifest của container chỉ định CPU request là
500 millicpu và CPU limit là 800 millicpu. Các giá trị này thỏa mãn ràng buộc CPU tối thiểu
và tối đa mà LimitRange áp đặt cho namespace này.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo
spec:
  containers:
  - name: constraints-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "500m"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod.yaml --namespace=constraints-cpu-example
```

Xác minh rằng Pod đang chạy và container của nó khỏe mạnh:

```shell
kubectl get pod constraints-cpu-demo --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-cpu-demo --output=yaml --namespace=constraints-cpu-example
```

Output cho thấy container duy nhất của Pod có CPU request là 500 millicpu và CPU limit là 800
millicpu. Các giá trị này thỏa mãn các ràng buộc mà LimitRange áp đặt.

```yaml
resources:
  limits:
    cpu: 800m
  requests:
    cpu: 500m
```

## Xóa Pod (Delete the Pod)

```shell
kubectl delete pod constraints-cpu-demo --namespace=constraints-cpu-example
```

## Thử tạo một Pod vượt quá ràng buộc CPU tối đa (Attempt to create a Pod that exceeds the maximum CPU constraint)

Đây là manifest cho một Pod có một container. Container này chỉ định CPU request là 500
millicpu và CPU limit là 1.5 cpu.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-2
spec:
  containers:
  - name: constraints-cpu-demo-2-ctr
    image: nginx
    resources:
      limits:
        cpu: "1.5"
      requests:
        cpu: "500m"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-2.yaml --namespace=constraints-cpu-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container không được chấp nhận.
Container đó không được chấp nhận vì nó chỉ định một CPU limit quá lớn:

```
Error from server (Forbidden): error when creating "examples/admin/resource/cpu-constraints-pod-2.yaml":
pods "constraints-cpu-demo-2" is forbidden: maximum cpu usage per Container is 800m, but limit is 1500m.
```

## Thử tạo một Pod không đạt mức CPU request tối thiểu (Attempt to create a Pod that does not meet the minimum CPU request)

Đây là manifest cho một Pod có một container. Container này chỉ định CPU request là 100
millicpu và CPU limit là 800 millicpu.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-3
spec:
  containers:
  - name: constraints-cpu-demo-3-ctr
    image: nginx
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "100m"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-3.yaml --namespace=constraints-cpu-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container không được chấp nhận.
Container đó không được chấp nhận vì nó chỉ định một CPU request thấp hơn mức tối thiểu đang
được thực thi:

```
Error from server (Forbidden): error when creating "examples/admin/resource/cpu-constraints-pod-3.yaml":
pods "constraints-cpu-demo-3" is forbidden: minimum cpu usage per Container is 200m, but request is 100m.
```

## Tạo một Pod không chỉ định bất kỳ CPU request hay limit nào (Create a Pod that does not specify any CPU request or limit)

Đây là manifest cho một Pod có một container. Container này không chỉ định CPU request, cũng
không chỉ định CPU limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-4
spec:
  containers:
  - name: constraints-cpu-demo-4-ctr
    image: vish/stress
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-4.yaml --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về Pod:

```
kubectl get pod constraints-cpu-demo-4 --namespace=constraints-cpu-example --output=yaml
```

Output cho thấy container duy nhất của Pod có CPU request là 800 millicpu và CPU limit là 800
millicpu. Làm thế nào container đó nhận được các giá trị này?

```yaml
resources:
  limits:
    cpu: 800m
  requests:
    cpu: 800m
```

Bởi vì container đó không chỉ định CPU request và limit của riêng nó, control plane đã áp
dụng
[CPU request và limit mặc định](230-cpu-default-namespace-vi.md)
từ LimitRange của namespace này.

Tại thời điểm này, Pod của bạn có thể đang chạy hoặc không. Hãy nhớ lại rằng một điều kiện
tiên quyết của tác vụ này là các Node của bạn phải có ít nhất 1 CPU khả dụng để sử dụng. Nếu
mỗi Node của bạn chỉ có 1 CPU, thì có thể không Node nào có đủ CPU cấp phát được (allocatable)
để đáp ứng một request 800 millicpu. Nếu bạn đang dùng các Node có 2 CPU, thì bạn có lẽ có đủ
CPU để đáp ứng request 800 millicpu.

Xóa Pod:

```
kubectl delete pod constraints-cpu-demo-4 --namespace=constraints-cpu-example
```

## Việc thực thi các ràng buộc CPU tối thiểu và tối đa (Enforcement of minimum and maximum CPU constraints)

Các ràng buộc CPU tối đa và tối thiểu mà một LimitRange áp đặt lên một namespace chỉ được
thực thi khi một Pod được tạo hoặc cập nhật. Nếu bạn thay đổi LimitRange, nó không ảnh hưởng
tới các Pod đã được tạo trước đó.

> **Ghi chú:**
>
> Khi sử dụng [thay đổi kích thước tài nguyên Pod tại chỗ (in-place Pod resize)](289-resize-container-resources-vi.md),
> các ràng buộc CPU cũng được thực thi. Nếu một lần resize làm cho các giá trị CPU của Pod vi
> phạm ràng buộc của LimitRange (vượt quá mức tối đa hoặc xuống dưới mức tối thiểu), lần
> resize đó sẽ bị từ chối và tài nguyên của Pod giữ nguyên các giá trị trước đó.

## Động cơ cho ràng buộc CPU tối thiểu và tối đa (Motivation for minimum and maximum CPU constraints)

Với vai trò người quản trị cluster, bạn có thể muốn áp đặt các hạn chế lên tài nguyên CPU mà
các Pod được sử dụng. Ví dụ:

* Mỗi Node trong cluster có 2 CPU. Bạn không muốn chấp nhận bất kỳ Pod nào yêu cầu nhiều hơn
  2 CPU, vì không Node nào trong cluster có thể đáp ứng request đó.

* Một cluster được chia sẻ giữa bộ phận production và bộ phận development của bạn. Bạn muốn
  cho phép các workload production tiêu thụ tới 3 CPU, nhưng muốn các workload development bị
  giới hạn ở 1 CPU. Bạn tạo các namespace riêng cho production và development, và áp dụng
  ràng buộc CPU cho từng namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace constraints-cpu-example
```

## Tiếp theo (What's next)

### Dành cho người quản trị cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình quota memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình quota Pod cho một Namespace](234-quota-pod-namespace-vi.md)

* [Cấu hình quota cho các API object](252-quota-api-object-vi.md)

### Dành cho lập trình viên ứng dụng (For app developers)

* [Gán tài nguyên memory cho container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. Pod vi phạm ràng buộc `min`/`max` bị chặn ở **thời điểm nào**, và điều gì xảy ra với các Pod
   đã chạy sẵn khi bạn sửa lại LimitRange của namespace?
2. Manifest LimitRange trong bài chỉ có `max` và `min`. Vậy vì sao
   `kubectl get limitrange --output=yaml` lại in ra hai field nữa, chúng tên gì và mang giá trị
   bao nhiêu?
3. **Câu bẫy.** Pod `constraints-cpu-demo-4` không khai CPU request lẫn CPU limit. Namespace lại
   đang thực thi `min: 200m`. Trực giác nói Pod này bị từ chối vì không có request nào để đem so
   với mức tối thiểu. Bài cho kết quả gì, và lập luận nào giải thích kết quả đó?
4. Hai thông báo `Error from server (Forbidden)` trong bài khác nhau ở chỗ nào? Chỉ rõ thông báo
   nào đối chiếu **limit** với `max` và thông báo nào đối chiếu **request** với `min`.
5. `lab-k8s-worker1` và `lab-k8s-worker2` mỗi node có 2 vCPU. Bài đặt điều kiện tiên quyết nào về
   CPU của Node, và nó dự đoán khác nhau thế nào giữa node 1 CPU và node 2 CPU khi Pod
   `constraints-cpu-demo-4` nhận request 800 millicpu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bị chặn **lúc tạo Pod** — đoạn mở đầu nói rõ: Pod không đáp ứng ràng buộc thì **không thể được
   tạo** trong namespace đó, và output là `Error from server (Forbidden)`. Mục *Việc thực thi các
   ràng buộc CPU tối thiểu và tối đa* bổ sung nửa còn lại: ràng buộc chỉ được thực thi **khi Pod
   được tạo hoặc cập nhật**, nên **sửa LimitRange không ảnh hưởng tới các Pod đã tạo trước đó** —
   chúng chạy tiếp với giá trị cũ.
2. Vì khi bạn khai `max`/`min` mà bỏ trống phần mặc định, Kubernetes **tự tạo** chúng: output hiện
   **`default: 800m` và `defaultRequest: 800m`**, tức **cả hai bằng đúng `max`**. Bài gọi thẳng
   hiện tượng này ra: mặc dù bạn không chỉ định các giá trị mặc định trong file cấu hình của
   LimitRange, chúng vẫn được tạo tự động.
3. Pod đó **được tạo bình thường**, và container của nó có `requests.cpu: 800m` cùng
   `limits.cpu: 800m`. Lập luận nằm ở ba bước mà control plane thực hiện: **bước gán mặc định chạy
   trước bước kiểm tra**. Container không tự khai gì nên nhận `default`/`defaultRequest` — vốn
   bằng `max` là 800m — rồi mới bị đem so với `min: 200m` và `max: 800m`, và nó **đạt cả hai**.
   Trực giác sai vì cho rằng không khai nghĩa là request bằng 0; thực tế không có container nào
   đi tới bước kiểm tra mà còn để trống.
4. Khác ở **ngưỡng bị đụng** và **field bị đem ra so**. `maximum cpu usage per Container is 800m,
   but limit is 1500m` — đụng **`max`**, và thứ bị so là **`limits.cpu`** của container.
   `minimum cpu usage per Container is 200m, but request is 100m` — đụng **`min`**, và thứ bị so
   là **`requests.cpu`**. Đọc đúng cặp *(ngưỡng, field)* là cách nhanh nhất biết phải sửa dòng nào
   trong manifest.
5. Điều kiện tiên quyết: **mỗi Node phải có ít nhất 1.0 CPU khả dụng cho các Pod**. Bài dự đoán:
   nếu mỗi Node **chỉ có 1 CPU** thì có thể **không Node nào đủ CPU cấp phát được** để đáp ứng
   request 800 millicpu, nên Pod tạo ra rồi vẫn có thể **không chạy**; còn với **Node 2 CPU** —
   đúng cấu hình hai worker của bạn — thì bài nói bạn có lẽ có đủ để đáp ứng request đó. Đây là
   ranh giới giữa **được chấp nhận** (LimitRange cho qua) và **chạy được** (Node còn chỗ), hai
   việc khác nhau.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
