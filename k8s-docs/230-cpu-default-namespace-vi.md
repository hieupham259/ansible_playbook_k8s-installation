# Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/>
>
> Định nghĩa giới hạn tài nguyên CPU mặc định cho một namespace, để mọi Pod mới
> trong namespace đó đều được cấu hình giới hạn tài nguyên CPU.

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

Trang này thuộc cặp thứ hai trong sáu trang con: **LimitRange đặt `default`/`defaultRequest`** —
**điền vào** Pod không khai gì. Đừng lẫn với cặp thứ nhất
([229](229-cpu-constraint-namespace-vi.md), [231](231-memory-constraint-namespace-vi.md)) là
LimitRange đặt `min`/`max` — **từ chối** Pod khai ngoài khoảng — và cặp thứ ba
([233](233-quota-memory-cpu-namespace-vi.md), [234](234-quota-pod-namespace-vi.md)) là
**ResourceQuota**, trần của **cả namespace**.

Việc "LimitRange chèn giá trị mặc định vào Pod trần" bạn đã làm ở
[Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) phần B2.3, và cái bẫy `default` nhỏ hơn
`requests` của client ở phần B2.6. Cái giai đoạn 25 thêm vào ở trang này là **hai trường hợp khai
nửa vời** — chỉ khai limit, hoặc chỉ khai request — và chúng cho ra hai kết quả khác nhau.

**Phải hiểu ở lần đọc này:**

- Pod **không khai gì**: nhận đúng hai giá trị của LimitRange — `requests.cpu: 500m` và
  `limits.cpu: "1"`, tức `defaultRequest: 0.5` và `default: 1` trong manifest LimitRange.
- Mục *Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request?*: request
  được đặt **bằng chính limit** của container, và bài nhấn mạnh container đó **không** nhận giá
  trị `defaultRequest` 0.5. Đây chính là "một số điều kiện nhất định" mà câu mở đầu trang bóng gió.
- Mục *Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit?*: request
  **giữ nguyên** giá trị bạn khai (`750m`), còn limit lấy `default` của namespace (`1`). Hai mục
  này đặt cạnh nhau cho thấy giá trị bạn tự khai luôn thắng, và phần thiếu được điền theo hai
  nguồn khác nhau tùy thiếu cái gì.
- Mục *Động lực cho CPU limit và request mặc định* nối trang này sang
  [233](233-quota-memory-cpu-namespace-vi.md): một CPU ResourceQuota buộc **mọi container trong
  namespace phải có CPU limit**, nên nếu không có LimitRange mặc định thì Pod nào quên khai limit
  sẽ không chạy được trong namespace đang bị quota giới hạn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ràng buộc thứ hai trong mục *Động lực* — cách một CPU ResourceQuota cộng dồn lượng đặt trước của mọi Pod và so với trần của namespace | trang này chỉ nêu để giải thích **vì sao cần giá trị mặc định**, không dạy cách đặt quota | bài [233](233-quota-memory-cpu-namespace-vi.md), trang con kế tiếp của bài 2/7 |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho quản trị viên cluster* và *Dành cho nhà phát triển ứng dụng* | là con trỏ, không có nội dung mới | các trang còn lại của bài 2/7 đọc ngay sau đây; nhánh nhà phát triển đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này hướng dẫn cách cấu hình CPU request và limit mặc định cho một namespace.

Một cluster Kubernetes có thể được chia thành nhiều namespace. Nếu bạn tạo một Pod trong một
namespace đã có CPU
[limit](110-manage-resources-containers-vi.md#requests-and-limits)
mặc định, và bất kỳ container nào trong Pod đó không chỉ định CPU limit của riêng nó, thì
control plane sẽ gán CPU limit mặc định cho container đó.

Kubernetes gán CPU request mặc định, nhưng chỉ trong một số điều kiện nhất định sẽ được
giải thích ở phần sau của trang này.

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

Nếu bạn chưa quen với ý nghĩa của 1.0 CPU trong Kubernetes, hãy đọc
[ý nghĩa của CPU](110-manage-resources-containers-vi.md#meaning-of-cpu).

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace default-cpu-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest cho một LimitRange ví dụ. Manifest này chỉ định một CPU request mặc định
và một CPU limit mặc định.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

Tạo LimitRange trong namespace default-cpu-example:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults.yaml --namespace=default-cpu-example
```

Bây giờ nếu bạn tạo một Pod trong namespace default-cpu-example, và bất kỳ container nào
trong Pod đó không chỉ định giá trị CPU request và CPU limit của riêng nó, thì control plane
sẽ áp dụng các giá trị mặc định: CPU request là 0.5 và CPU limit mặc định là 1.

Dưới đây là manifest cho một Pod có một container. Container này không chỉ định
CPU request và limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo
spec:
  containers:
  - name: default-cpu-demo-ctr
    image: nginx
```

Tạo Pod.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod.yaml --namespace=default-cpu-example
```

Xem specification của Pod:

```shell
kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
```

Output cho thấy container duy nhất của Pod có CPU request là 500m `cpu`
(bạn có thể đọc là "500 millicpu"), và CPU limit là 1 `cpu`.
Đây là các giá trị mặc định do LimitRange chỉ định.

```shell
containers:
- image: nginx
  imagePullPolicy: Always
  name: default-cpu-demo-ctr
  resources:
    limits:
      cpu: "1"
    requests:
      cpu: 500m
```

## Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request? (What if you specify a container's limit, but not its request?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định CPU limit,
nhưng không chỉ định request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-2
spec:
  containers:
  - name: default-cpu-demo-2-ctr
    image: nginx
    resources:
      limits:
        cpu: "1"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod-2.yaml --namespace=default-cpu-example
```

Xem [specification](16-working-with-objects-vi.md#object-spec-and-status)
của Pod bạn vừa tạo:

```
kubectl get pod default-cpu-demo-2 --output=yaml --namespace=default-cpu-example
```

Output cho thấy CPU request của container được đặt bằng với CPU limit của nó.
Lưu ý rằng container không được gán giá trị CPU request mặc định 0.5 `cpu`:

```
resources:
  limits:
    cpu: "1"
  requests:
    cpu: "1"
```

## Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit? (What if you specify a container's request, but not its limit?)

Dưới đây là manifest ví dụ cho một Pod có một container. Container này chỉ định CPU request,
nhưng không chỉ định limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-3
spec:
  containers:
  - name: default-cpu-demo-3-ctr
    image: nginx
    resources:
      requests:
        cpu: "0.75"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod-3.yaml --namespace=default-cpu-example
```

Xem specification của Pod bạn vừa tạo:

```
kubectl get pod default-cpu-demo-3 --output=yaml --namespace=default-cpu-example
```

Output cho thấy CPU request của container được đặt theo giá trị bạn đã chỉ định lúc tạo Pod
(nói cách khác: nó khớp với manifest). Tuy nhiên, CPU limit của chính container đó được đặt
là 1 `cpu`, chính là CPU limit mặc định của namespace này.

```
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 750m
```

## Động lực cho CPU limit và request mặc định (Motivation for default CPU limits and requests)

Nếu namespace của bạn đã cấu hình hạn ngạch tài nguyên (resource quota) cho CPU,
thì việc có sẵn một giá trị mặc định cho CPU limit là rất hữu ích.
Dưới đây là hai trong số các ràng buộc mà một hạn ngạch tài nguyên CPU áp đặt lên một namespace:

* Với mọi Pod chạy trong namespace, mỗi container của nó phải có CPU limit.
* CPU limit áp dụng một lượng tài nguyên được đặt trước (resource reservation) trên node
  mà Pod đó được lập lịch (schedule). Tổng lượng CPU được đặt trước cho tất cả các Pod
  trong namespace không được vượt quá một giới hạn đã chỉ định.

Khi bạn thêm một LimitRange:

Nếu bất kỳ Pod nào trong namespace đó chứa một container không chỉ định CPU limit của riêng nó,
control plane sẽ áp dụng CPU limit mặc định cho container đó, và Pod có thể được phép chạy
trong một namespace đang bị giới hạn bởi một CPU ResourceQuota.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace default-cpu-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

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

1. LimitRange `cpu-limit-range` khai `default: cpu 1` và `defaultRequest: cpu 0.5`. Pod
   `default-cpu-demo` không khai gì cả thì `resources` cuối cùng của container là bao nhiêu, và
   mỗi con số lấy từ field nào của LimitRange?
2. **Câu bẫy.** Pod `default-cpu-demo-2` khai `limits.cpu: "1"` nhưng bỏ trống request, trong khi
   namespace có sẵn `defaultRequest: 0.5`. Request cuối cùng của container là bao nhiêu? Vì sao
   suy luận "thiếu request thì lấy `defaultRequest`" lại sai đúng ở trường hợp này?
3. Pod `default-cpu-demo-3` làm ngược lại: khai `requests.cpu: "0.75"`, bỏ trống limit. Hai giá
   trị cuối cùng là gì? Đặt cạnh câu 2, rút ra quy tắc chung nào về thứ tự ưu tiên giữa giá trị
   bạn khai và giá trị mặc định của namespace?
4. Trên cluster lab (hai worker, mỗi node 2 vCPU), bạn định cấp một namespace cho một nhóm và đặt
   CPU ResourceQuota cho namespace đó. Theo mục *Động lực*, vì sao vẫn phải kèm một LimitRange
   mặc định, và ràng buộc nào của quota làm điều đó thành bắt buộc?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Container nhận **`requests.cpu: 500m` và `limits.cpu: "1"`**. `500m` đến từ **`defaultRequest`**
   (`0.5` cpu, đọc là 500 millicpu) và `1` đến từ **`default`**. Đây là trường hợp đơn giản nhất
   và cũng là trường hợp duy nhất trong bài mà cả hai giá trị đều do LimitRange cấp.
2. Request cuối cùng là **`"1"`, tức bằng chính limit** — bài nói thẳng: CPU request của container
   **được đặt bằng với CPU limit của nó**, và **container không được gán giá trị CPU request mặc
   định 0.5 cpu**. Suy luận kia sai vì `defaultRequest` không phải một cái lưới hứng mọi trường
   hợp thiếu request; câu mở đầu trang đã báo trước điều đó — Kubernetes gán CPU request mặc định
   **nhưng chỉ trong một số điều kiện nhất định**. Khi container đã tự khai limit, điều kiện đó
   không còn.
3. Container giữ **`requests.cpu: 750m`** đúng như manifest, và nhận **`limits.cpu: "1"`** — chính
   là CPU limit mặc định của namespace. Quy tắc chung: **giá trị bạn tự khai luôn được giữ
   nguyên**, phần còn thiếu mới được điền; nhưng nguồn để điền **không giống nhau** — thiếu limit
   thì lấy `default` của LimitRange, còn thiếu request khi đã có limit thì lấy **chính limit đó**,
   không lấy `defaultRequest`.
4. Vì một CPU ResourceQuota áp đặt ràng buộc: **mọi Pod chạy trong namespace phải có CPU limit ở
   từng container**. Nếu nhóm đó gửi lên một Pod quên khai limit, Pod sẽ không được phép chạy
   trong namespace có quota. Thêm LimitRange mặc định thì **control plane áp CPU limit mặc định
   cho container thiếu**, và Pod đi qua được quota. Nói gọn: quota đặt ra yêu cầu, LimitRange lo
   phần đáp ứng yêu cầu đó thay cho người dùng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
