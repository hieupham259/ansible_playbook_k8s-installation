# Cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/>
>
> Định nghĩa một khoảng giá trị hợp lệ cho giới hạn tài nguyên bộ nhớ trong một namespace,
> để mọi Pod mới trong namespace đó đều nằm trong khoảng bạn cấu hình.

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

Trang này là bản memory của [229](229-cpu-constraint-namespace-vi.md): cùng cặp thứ nhất trong
sáu trang con — **LimitRange đặt `min`/`max`**, **từ chối** Pod khai ngoài khoảng. Đọc nhanh phần
trùng khuôn, và dừng lại ở ba thứ mà trang CPU không có: **đơn vị bộ nhớ bị chuẩn hóa** trong
thông báo từ chối, cách đọc bằng `kubectl describe`, và điều kiện tiên quyết tính theo GiB.
[Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) phần B2.4 và B2.5 đã thực hành cơ chế từ
chối này; phần B2.7 đã thực hành việc sửa LimitRange không đụng Pod đang chạy.

**Phải hiểu ở lần đọc này:**

- Cùng hiện tượng như bài [229](229-cpu-constraint-namespace-vi.md): manifest chỉ khai
  `max: 1Gi` và `min: 500Mi`, nhưng `kubectl get limitrange --output=yaml` in ra thêm
  **`default: 1Gi` và `defaultRequest: 1Gi`** được tạo tự động — cả hai bám theo **`max`**, không
  bám theo `min`.
- Ba việc Kubernetes làm với mỗi Pod trong namespace: gán request/limit **mặc định** cho container
  không tự khai → xác minh mọi container request **ít nhất 500 MiB** → xác minh mọi container
  không vượt **1024 MiB (1 GiB)**.
- Hai thông báo `Error from server (Forbidden)` và **cách đơn vị được chuẩn hóa** khi in ra:
  `maximum memory usage per Container is 1Gi, but limit is 1536Mi` (bạn khai `1.5Gi`, thông báo
  in `1536Mi`) và `minimum memory usage per Container is 500Mi, but request is 100Mi`.
- Pod `constraints-mem-demo-4` không khai gì **vẫn được tạo** và nhận `1Gi/1Gi` từ
  [memory request và limit mặc định](232-memory-default-namespace-vi.md) sinh tự động. Trang này
  thêm một cách đọc mà trang CPU không có: `kubectl describe pod` rồi tìm mục `Requests:`. Kèm
  điều kiện tiên quyết — Node phải có ít nhất **1 GiB** bộ nhớ, và bài nói rõ Node 2 GiB thì có lẽ
  đủ chỗ cho request 1 GiB.
- Mục *Thực thi các ràng buộc bộ nhớ tối thiểu và tối đa* và mục *Động lực*: ràng buộc chỉ được
  thực thi **khi Pod được tạo hoặc cập nhật**, sửa LimitRange **không** đụng Pod đã tạo; và lý do
  đặt ràng buộc là hai tình huống rất đúng với giai đoạn 25 — chặn Pod xin nhiều hơn dung lượng
  một Node có thể đáp ứng, và **tách namespace production với development** rồi áp ràng buộc khác
  nhau cho từng namespace.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú về **in-place Pod resize** và việc resize bị từ chối khi vi phạm ràng buộc bộ nhớ | resize tại chỗ là một bài riêng, không phải nội dung của LimitRange | bài [289](289-resize-container-resources-vi.md), dòng Thực hành của [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho quản trị viên cluster* và *Dành cho nhà phát triển ứng dụng* | là con trỏ, không có nội dung mới | các trang còn lại của bài 2/7 đọc ngay sau đây; nhánh nhà phát triển đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này hướng dẫn cách đặt giá trị tối thiểu và tối đa cho lượng bộ nhớ mà các container
chạy trong một namespace được sử dụng. Bạn chỉ định các giá trị bộ nhớ tối thiểu và tối đa
trong một object
[LimitRange](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/).
Nếu một Pod không thỏa mãn các ràng buộc mà LimitRange áp đặt, nó sẽ không thể được tạo
trong namespace đó.

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

Mỗi node trong cluster của bạn phải có ít nhất 1 GiB bộ nhớ khả dụng cho các Pod.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace constraints-mem-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest ví dụ cho một LimitRange:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-min-max-demo-lr
spec:
  limits:
  - max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```

Tạo LimitRange:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints.yaml --namespace=constraints-mem-example
```

Xem thông tin chi tiết về LimitRange:

```shell
kubectl get limitrange mem-min-max-demo-lr --namespace=constraints-mem-example --output=yaml
```

Output hiển thị các ràng buộc bộ nhớ tối thiểu và tối đa đúng như mong đợi. Nhưng hãy để ý
rằng mặc dù bạn không chỉ định giá trị mặc định trong file cấu hình của LimitRange,
chúng vẫn được tạo tự động.

```
  limits:
  - default:
      memory: 1Gi
    defaultRequest:
      memory: 1Gi
    max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```

Bây giờ, mỗi khi bạn định nghĩa một Pod trong namespace constraints-mem-example, Kubernetes
sẽ thực hiện các bước sau:

* Nếu bất kỳ container nào trong Pod đó không chỉ định memory request và limit của riêng nó,
  control plane sẽ gán memory request và limit mặc định cho container đó.

* Xác minh rằng mọi container trong Pod đó yêu cầu (request) ít nhất 500 MiB bộ nhớ.

* Xác minh rằng mọi container trong Pod đó yêu cầu không quá 1024 MiB (1 GiB) bộ nhớ.

Dưới đây là manifest cho một Pod có một container. Trong spec của Pod, container duy nhất
chỉ định memory request là 600 MiB và memory limit là 800 MiB. Các giá trị này thỏa mãn
ràng buộc bộ nhớ tối thiểu và tối đa mà LimitRange áp đặt.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo
spec:
  containers:
  - name: constraints-mem-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
      requests:
        memory: "600Mi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod.yaml --namespace=constraints-mem-example
```

Xác minh rằng Pod đang chạy và container của nó hoạt động bình thường:

```shell
kubectl get pod constraints-mem-demo --namespace=constraints-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-mem-demo --output=yaml --namespace=constraints-mem-example
```

Output cho thấy container trong Pod đó có memory request là 600 MiB và memory limit là 800 MiB.
Các giá trị này thỏa mãn các ràng buộc mà LimitRange áp đặt cho namespace này:

```yaml
resources:
  limits:
     memory: 800Mi
  requests:
    memory: 600Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod constraints-mem-demo --namespace=constraints-mem-example
```

## Thử tạo một Pod vượt quá ràng buộc bộ nhớ tối đa (Attempt to create a Pod that exceeds the maximum memory constraint)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory request
là 800 MiB và memory limit là 1.5 GiB.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-2
spec:
  containers:
  - name: constraints-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1.5Gi"
      requests:
        memory: "800Mi"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-2.yaml --namespace=constraints-mem-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container yêu cầu nhiều bộ nhớ hơn
mức cho phép:

```
Error from server (Forbidden): error when creating "examples/admin/resource/memory-constraints-pod-2.yaml":
pods "constraints-mem-demo-2" is forbidden: maximum memory usage per Container is 1Gi, but limit is 1536Mi.
```

## Thử tạo một Pod không đạt mức memory request tối thiểu (Attempt to create a Pod that does not meet the minimum memory request)

Dưới đây là manifest cho một Pod có một container. Container đó chỉ định memory request
là 100 MiB và memory limit là 800 MiB.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-3
spec:
  containers:
  - name: constraints-mem-demo-3-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
      requests:
        memory: "100Mi"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-3.yaml --namespace=constraints-mem-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container yêu cầu ít bộ nhớ hơn
mức tối thiểu bắt buộc:

```
Error from server (Forbidden): error when creating "examples/admin/resource/memory-constraints-pod-3.yaml":
pods "constraints-mem-demo-3" is forbidden: minimum memory usage per Container is 500Mi, but request is 100Mi.
```

## Tạo một Pod không chỉ định memory request hay limit nào (Create a Pod that does not specify any memory request or limit)

Dưới đây là manifest cho một Pod có một container. Container này không chỉ định memory request,
và cũng không chỉ định memory limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-4
spec:
  containers:
  - name: constraints-mem-demo-4-ctr
    image: nginx
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-4.yaml --namespace=constraints-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-mem-demo-4 --namespace=constraints-mem-example --output=yaml
```

Output cho thấy container duy nhất của Pod có memory request là 1 GiB và memory limit là 1 GiB.
Làm thế nào container đó có được những giá trị này?

```
resources:
  limits:
    memory: 1Gi
  requests:
    memory: 1Gi
```

Vì Pod của bạn không định nghĩa memory request và limit nào cho container đó, cluster đã
áp dụng
[memory request và limit mặc định](232-memory-default-namespace-vi.md)
từ LimitRange.

Điều này có nghĩa là định nghĩa của Pod đó sẽ hiển thị các giá trị này. Bạn có thể kiểm tra
bằng `kubectl describe`:

```shell
# Tìm mục "Requests:" trong output
kubectl describe pod constraints-mem-demo-4 --namespace=constraints-mem-example
```

Tại thời điểm này, Pod của bạn có thể đang chạy hoặc không chạy. Hãy nhớ rằng điều kiện
tiên quyết của bài này là các Node của bạn có ít nhất 1 GiB bộ nhớ. Nếu mỗi Node của bạn chỉ
có 1 GiB bộ nhớ, thì không Node nào có đủ bộ nhớ cấp phát được (allocatable) để đáp ứng một
memory request 1 GiB. Nếu bạn đang dùng các Node có 2 GiB bộ nhớ, thì có lẽ bạn có đủ chỗ
để đáp ứng request 1 GiB đó.

Xóa Pod của bạn:

```shell
kubectl delete pod constraints-mem-demo-4 --namespace=constraints-mem-example
```

## Thực thi các ràng buộc bộ nhớ tối thiểu và tối đa (Enforcement of minimum and maximum memory constraints)

Các ràng buộc bộ nhớ tối đa và tối thiểu mà một LimitRange áp đặt lên một namespace chỉ được
thực thi khi một Pod được tạo hoặc cập nhật. Nếu bạn thay đổi LimitRange, điều đó không ảnh
hưởng đến các Pod đã được tạo trước đó.

> **Ghi chú:**
> Khi sử dụng [thay đổi kích thước Pod tại chỗ (in-place Pod resize)](289-resize-container-resources-vi.md),
> các ràng buộc bộ nhớ cũng được thực thi. Nếu một lần resize khiến các giá trị bộ nhớ của Pod
> vi phạm ràng buộc của LimitRange (vượt quá mức tối đa hoặc thấp hơn mức tối thiểu),
> lần resize đó sẽ bị từ chối và tài nguyên của Pod giữ nguyên các giá trị trước đó.

## Động lực cho ràng buộc bộ nhớ tối thiểu và tối đa (Motivation for minimum and maximum memory constraints)

Với vai trò quản trị viên cluster, bạn có thể muốn áp đặt các giới hạn lên lượng bộ nhớ mà
các Pod được sử dụng. Ví dụ:

* Mỗi Node trong cluster có 2 GiB bộ nhớ. Bạn không muốn chấp nhận bất kỳ Pod nào yêu cầu
  hơn 2 GiB bộ nhớ, vì không Node nào trong cluster có thể đáp ứng yêu cầu đó.

* Một cluster được dùng chung bởi bộ phận production và bộ phận development của bạn.
  Bạn muốn cho phép các workload production sử dụng tối đa 8 GiB bộ nhớ, nhưng muốn giới hạn
  các workload development ở mức 512 MiB. Bạn tạo các namespace riêng cho production và
  development, và áp dụng ràng buộc bộ nhớ cho từng namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace constraints-mem-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

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

1. Bạn khai LimitRange chỉ có `max: 1Gi` và `min: 500Mi`, không khai gì thêm. Đọc lại bằng
   `kubectl get limitrange --output=yaml` thì thấy thêm hai field. Chúng tên gì, mang giá trị bao
   nhiêu, và giá trị đó bám theo `min` hay theo `max`?
2. **Câu bẫy.** Manifest của bạn ghi `limits.memory: "1.5Gi"`, nhưng thông báo từ chối lại viết
   `limit is 1536Mi`. Có phải API server đọc sai manifest không? Trong câu
   `maximum memory usage per Container is 1Gi, but limit is 1536Mi`, con số nào là **ngưỡng** và
   con số nào là **giá trị bạn khai**?
3. Hai worker của bạn — `lab-k8s-worker1` và `lab-k8s-worker2` — mỗi node có 6 GB RAM. Bài đặt
   điều kiện tiên quyết nào về bộ nhớ của Node, và vì sao Pod `constraints-mem-demo-4` với memory
   request 1 GiB không rơi vào tình huống mà bài cảnh báo?
4. Mục *Động lực* nêu hai lý do đặt ràng buộc `min`/`max`. Lý do thứ hai áp dụng thế nào khi một
   cluster phải phục vụ cả bộ phận production lẫn bộ phận development?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Hai field là **`default` và `defaultRequest`**, và **cả hai đều bằng `1Gi`** — tức bám theo
   **`max`**, không bám theo `min`. Bài nói thẳng: mặc dù bạn không chỉ định giá trị mặc định
   trong file cấu hình của LimitRange, chúng vẫn được tạo tự động. Hệ quả thực tế: một namespace
   chỉ khai `min`/`max` vẫn ngầm ép mọi Pod trần lên **mức trần** của khoảng.
2. **Không đọc sai.** `1.5Gi` và `1536Mi` là **cùng một lượng bộ nhớ**, chỉ khác đơn vị in ra;
   thông báo chuẩn hóa về `Mi` nên nhìn lạ mắt chứ không phải sai. Trong câu đó, **`1Gi` là
   ngưỡng** — chính là `max` của LimitRange — còn **`1536Mi` là giá trị `limits.memory` bạn khai**.
   Chỗ dễ mất thời gian là đi sửa manifest theo con số `1536Mi` thay vì hiểu rằng phải hạ limit
   xuống không quá `1Gi`.
3. Điều kiện tiên quyết: **mỗi Node có ít nhất 1 GiB bộ nhớ**. Bài cảnh báo trường hợp Node **chỉ
   có 1 GiB**: khi đó không Node nào còn đủ bộ nhớ **cấp phát được (allocatable)** để đáp ứng
   request 1 GiB, nên Pod tạo ra rồi vẫn có thể **không chạy**. Hai worker 6 GB của bạn dư xa
   ngưỡng đó — bài còn nói Node 2 GiB đã "có lẽ đủ" — nên Pod này được tạo **và** lập lịch được.
   Vẫn phải giữ phân biệt: LimitRange quyết định Pod **có được chấp nhận** hay không, còn bộ nhớ
   allocatable của Node quyết định nó **có chạy được** hay không.
4. Lý do thứ hai là **chia cluster dùng chung theo namespace**: tạo namespace riêng cho production
   và cho development, rồi **áp ràng buộc bộ nhớ khác nhau cho từng namespace** — ví dụ của bài là
   cho phép production dùng tới 8 GiB còn giới hạn development ở 512 MiB. Đây đúng là mô hình mà
   giai đoạn 25 nhắm tới: cùng một cluster, nhiều nhóm, và trần khác nhau đặt ở ranh giới
   namespace chứ không ở từng Pod rời rạc.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
