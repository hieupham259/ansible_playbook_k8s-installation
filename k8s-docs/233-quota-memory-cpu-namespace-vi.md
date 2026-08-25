# Cấu hình Quota Memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/
>
> Định nghĩa giới hạn tổng tài nguyên memory và CPU cho một namespace.

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

Đây là trang mở cặp thứ ba trong sáu trang con — **ResourceQuota**, tức **trần của cả
namespace**. Bốn trang trước đều là LimitRange và tác động lên **từng Pod hoặc từng container**:
[229](229-cpu-constraint-namespace-vi.md), [231](231-memory-constraint-namespace-vi.md) đặt
`min`/`max` để **từ chối**; [230](230-cpu-default-namespace-vi.md),
[232](232-memory-default-namespace-vi.md) đặt `default`/`defaultRequest` để **điền vào**. Ranh
giới quota với LimitRange chính là thứ **Checkpoint giai đoạn 25** bắt bạn giải thích, và trang
này nói thẳng ra ở mục *Thảo luận*.

Bạn đã đọc lý thuyết ở bài [134](134-resource-quotas-vi.md) và thực hành ở
[Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md): phần B3 làm trần tổng CPU/memory, so `used`
với `hard` và hai kiểu từ chối; phần B5 làm việc quota **buộc mọi Pod phải khai** tài nguyên. Cái
giai đoạn 25 thêm vào ở trang này là **cấu trúc ba mảnh của thông báo từ chối** — thứ Checkpoint
bắt bạn đọc đúng.

**Phải hiểu ở lần đọc này:**

- `spec.hard` của ResourceQuota `mem-cpu-demo` có bốn khóa — `requests.cpu`, `requests.memory`,
  `limits.cpu`, `limits.memory` — nhưng bài liệt kê **năm** yêu cầu áp lên namespace. Yêu cầu đầu
  tiên không phải phép cộng nào cả: **mọi container của mọi Pod phải có đủ memory request, memory
  limit, cpu request và cpu limit**. Đây đúng là ràng buộc mà [230](230-cpu-default-namespace-vi.md)
  và [232](232-memory-default-namespace-vi.md) viện dẫn khi khuyên kèm LimitRange mặc định.
- Bốn yêu cầu còn lại đều là **tổng của tất cả Pod trong namespace**: tổng memory request ≤ 1 GiB,
  tổng memory limit ≤ 2 GiB, tổng CPU request ≤ 1 cpu, tổng CPU limit ≤ 2 cpu.
- Đọc mức tiêu thụ bằng `kubectl get resourcequota mem-cpu-demo --output=yaml`: `status.hard` là
  trần, `status.used` là phần đã dùng. Sau khi Pod đầu chạy, `used` bằng đúng request và limit
  của chính Pod đó.
- Thông báo từ chối Pod thứ hai có **ba mảnh** phải đọc tách bạch: `requested:` (cái Pod mới xin),
  `used:` (đang dùng), `limited:` (trần) — và phép tính bị vỡ là **600 MiB + 700 MiB > 1 GiB**.
- Mục *Thảo luận* chốt ranh giới của cả giai đoạn 25: ResourceQuota giới hạn **tổng mức sử dụng
  trong một namespace**; muốn giới hạn **từng Pod riêng lẻ hoặc các container trong Pod** thì dùng
  [LimitRange](133-limit-range-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mẹo dùng `jq` cùng JSONPath để in riêng `status.used` | chính bài ghi đây là tùy chọn — "nếu bạn có công cụ `jq`" — và lộ trình không cài thêm phần mềm vào cluster lab; `--output=yaml` đã đọc được đủ | không cần cho lộ trình; cách đọc bằng `--output=yaml` nằm ngay trong bài và đã dùng ở [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) phần B3 |
| Ghi chú về **in-place Pod resize** — quota được thực thi trên giá trị sau khi resize | resize tại chỗ là một bài riêng | bài [289](289-resize-container-resources-vi.md), dòng Thực hành của [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho quản trị viên cluster* và *Dành cho nhà phát triển ứng dụng* | là con trỏ, không có nội dung mới | trang con cuối của bài 2/7 là [234](234-quota-pod-namespace-vi.md), đọc ngay sau đây; nhánh nhà phát triển đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này hướng dẫn cách đặt hạn ngạch (quota) cho tổng lượng memory và CPU mà tất cả các Pod
đang chạy trong một namespace được phép sử dụng. Bạn chỉ định quota trong một đối tượng
[ResourceQuota](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Bạn phải có quyền tạo namespace trong cluster của mình.

Mỗi node trong cluster của bạn phải có ít nhất 1 GiB memory.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace quota-mem-cpu-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là manifest cho một ResourceQuota mẫu:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu.yaml --namespace=quota-mem-cpu-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example --output=yaml
```

ResourceQuota này đặt các yêu cầu sau lên namespace quota-mem-cpu-example:

* Với mọi Pod trong namespace, mỗi container phải có memory request, memory limit, cpu request
  và cpu limit.
* Tổng memory request của tất cả các Pod trong namespace đó không được vượt quá 1 GiB.
* Tổng memory limit của tất cả các Pod trong namespace đó không được vượt quá 2 GiB.
* Tổng CPU request của tất cả các Pod trong namespace đó không được vượt quá 1 cpu.
* Tổng CPU limit của tất cả các Pod trong namespace đó không được vượt quá 2 cpu.

Xem [ý nghĩa của CPU](110-manage-resources-containers-vi.md#meaning-of-cpu)
để hiểu Kubernetes định nghĩa "1 CPU" như thế nào.

## Tạo một Pod (Create a Pod)

Dưới đây là manifest cho một Pod mẫu:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu-pod.yaml --namespace=quota-mem-cpu-example
```

Xác minh rằng Pod đang chạy và container (duy nhất) của nó khỏe mạnh:

```shell
kubectl get pod quota-mem-cpu-demo --namespace=quota-mem-cpu-example
```

Một lần nữa, xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example --output=yaml
```

Kết quả hiển thị quota cùng với lượng quota đã được sử dụng. Bạn có thể thấy các request và
limit về memory và CPU của Pod không vượt quá quota.

```
status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

Nếu bạn có công cụ `jq`, bạn cũng có thể truy vấn (bằng
[JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)) để lấy riêng các giá trị
`used`, **và** in phần kết quả đó ra một cách dễ đọc. Ví dụ:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example -o jsonpath='{ .status.used }' | jq .
```

## Thử tạo Pod thứ hai (Attempt to create a second Pod)

Dưới đây là manifest cho Pod thứ hai:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo-2
spec:
  containers:
  - name: quota-mem-cpu-demo-2-ctr
    image: redis
    resources:
      limits:
        memory: "1Gi"
        cpu: "800m"
      requests:
        memory: "700Mi"
        cpu: "400m"
```

Trong manifest, bạn có thể thấy Pod này có memory request là 700 MiB. Lưu ý rằng tổng của
memory request đã dùng cộng với memory request mới này vượt quá quota memory request:
600 MiB + 700 MiB > 1 GiB.

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu-pod-2.yaml --namespace=quota-mem-cpu-example
```

Pod thứ hai không được tạo. Kết quả cho thấy việc tạo Pod thứ hai sẽ khiến tổng memory request
vượt quá quota memory request.

```
Error from server (Forbidden): error when creating "examples/admin/resource/quota-mem-cpu-pod-2.yaml":
pods "quota-mem-cpu-demo-2" is forbidden: exceeded quota: mem-cpu-demo,
requested: requests.memory=700Mi,used: requests.memory=600Mi, limited: requests.memory=1Gi
```

## Thảo luận (Discussion)

Như bạn đã thấy trong bài thực hành này, bạn có thể dùng ResourceQuota để giới hạn tổng memory
request của tất cả các Pod đang chạy trong một namespace. Bạn cũng có thể giới hạn tổng của
memory limit, cpu request và cpu limit.

Thay vì quản lý tổng mức sử dụng tài nguyên trong một namespace, có thể bạn muốn giới hạn từng
Pod riêng lẻ, hoặc các container trong những Pod đó. Để đạt được kiểu giới hạn như vậy, hãy dùng
[LimitRange](133-limit-range-vi.md).

> **Ghi chú:**
>
> Khi sử dụng [thay đổi kích thước Pod tại chỗ (in-place Pod resize)](289-resize-container-resources-vi.md),
> việc thực thi ResourceQuota được áp dụng lên các giá trị sau khi thay đổi. Nếu một lần thay
> đổi kích thước khiến namespace vượt quá giới hạn quota, thao tác đó bị từ chối và tài nguyên
> của Pod giữ nguyên không đổi.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-mem-cpu-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota số lượng Pod cho một Namespace](234-quota-pod-namespace-vi.md)

* [Cấu hình Quota cho các đối tượng API](252-quota-api-object-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở mức Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. `spec.hard` chỉ có bốn dòng, nhưng bài nói ResourceQuota này đặt **năm** yêu cầu lên namespace.
   Yêu cầu nào **không** phải một phép cộng, và nó buộc người dùng namespace phải làm gì?
2. **Câu bẫy.** Pod `quota-mem-cpu-demo-2` xin `requests.memory: 700Mi`, `limits.memory: 1Gi`,
   `requests.cpu: 400m`, `limits.cpu: 800m` — **không con số nào vượt bất kỳ trần nào** trong
   `hard`. Vậy vì sao nó bị từ chối?
3. Thông báo từ chối chứa ba mảnh `requested:`, `used:` và `limited:`. Mỗi mảnh là con số của ai,
   và bạn dùng chúng thế nào để biết phải sửa gì?
4. Trên cluster lab, hai worker mỗi node 2 vCPU và 6 GB RAM. Theo mục *Thảo luận*, giữa
   ResourceQuota và LimitRange thì cái nào chặn được **một Pod khổng lồ** và cái nào chặn được
   **quá nhiều Pod cỡ vừa**? Vì sao Checkpoint giai đoạn 25 bắt phân biệt đúng hai thứ này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Yêu cầu đầu tiên: **với mọi Pod trong namespace, mỗi container phải có memory request, memory
   limit, cpu request và cpu limit**. Nó không cộng gì cả — nó bắt buộc **khai báo**. Hệ quả: chỉ
   cần một container để trống một trong bốn giá trị là Pod bị chặn, và đó chính là lý do bài
   [230](230-cpu-default-namespace-vi.md) và [232](232-memory-default-namespace-vi.md) khuyên kèm
   một LimitRange mặc định để control plane điền hộ.
2. Vì quota **không đo từng Pod, nó đo tổng của cả namespace**. Pod đầu tiên đã chiếm
   `requests.memory: 600Mi`; cộng thêm `700Mi` của Pod mới thành **1300 MiB, vượt trần
   `requests.memory: 1Gi`**. Chỗ dễ sai là đem từng con số của Pod so với từng con số trong `hard`
   rồi kết luận "hợp lệ" — phép so đúng là **`used` + `requested` so với `limited`**.
3. `requested:` là **cái Pod mới đang xin** (`requests.memory=700Mi`); `used:` là **phần namespace
   đang dùng** (`requests.memory=600Mi`); `limited:` là **trần trong `hard`** (`requests.memory=1Gi`).
   Cách dùng: `limited` trừ `used` cho biết **còn bao nhiêu chỗ trống**, và `requested` cho biết
   Pod mới cần bao nhiêu — chênh lệch quyết định bạn phải **hạ request của Pod mới**, **xóa bớt
   Pod cũ**, hay **nâng quota**. Tên khóa in trong thông báo (`requests.memory`) còn chỉ đúng dòng
   nào trong `hard` đang bị đụng.
4. **LimitRange chặn một Pod khổng lồ** — nó đặt `min`/`max` cho **từng container**, nên một Pod
   xin quá nhiều bị từ chối ngay dù namespace còn trống. **ResourceQuota chặn quá nhiều Pod cỡ
   vừa** — mỗi Pod đều hợp lệ, nhưng **tổng** của chúng đụng trần namespace, đúng như Pod thứ hai
   trong bài. Mục *Thảo luận* nói thẳng: quota quản **tổng mức sử dụng trong namespace**, còn muốn
   giới hạn **từng Pod hoặc từng container** thì dùng LimitRange. Checkpoint bắt phân biệt vì hai
   cơ chế này hỏng theo hai kiểu khác nhau và thông báo từ chối cũng khác nhau — nhầm chúng là
   sửa sai chỗ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
