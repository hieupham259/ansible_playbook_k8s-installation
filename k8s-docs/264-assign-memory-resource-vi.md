# Gán tài nguyên memory cho Container và Pod (Assign Memory Resources to Containers and Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 2/9 ·
Kiểm chứng ở [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) phần B1.2, B2.2 và B4.1.

Bài này là **bản song sinh** của bài [263](263-assign-cpu-resource-vi.md): cùng bố cục, cùng thứ
tự mục, nhưng cho memory. Đọc để tìm chỗ hai bài **không** giống nhau — vượt limit CPU thì bị điều
tiết, còn vượt limit memory thì bị **chấm dứt**. Đó là ranh giới quan trọng nhất của cả nhóm 3c.

Bài chạy `kubectl top` ở vài bước; cluster lab chưa có metrics-server nên đọc để hiểu con số, đừng
chờ chạy được lệnh đó.

**Phải hiểu ở lần đọc này:**

- Mục *Vượt quá memory limit của Container*: container **được phép** vượt memory **request** nếu
  Node còn memory rảnh, nhưng vượt **limit** thì thành ứng viên bị chấm dứt và sẽ bị chấm dứt nếu
  tiếp tục tiêu thụ. Container chấm dứt được khởi động lại thì kubelet khởi động lại nó, y như với
  mọi lỗi runtime khác.
- Dấu vết của một lần bị kill, cũng ở mục đó: `STATUS` là `OOMKilled`, `lastState.terminated` ghi
  `reason: OOMKilled` với `exitCode: 137`, cột `RESTARTS` tăng dần, `kubectl describe pod` cho
  `BackOff restarting failed container`, và `kubectl describe nodes` ghi
  `Memory cgroup out of memory`.
- Mục *Chỉ định một memory request quá lớn so với các Node của bạn*: request/limit của Pod là
  **tổng** của các container; lập lịch dựa trên **request**; request `1000Gi` khiến Pod ở
  `Pending` vô thời hạn với `FailedScheduling ... Insufficient memory`.
- Mục *Đơn vị memory*: memory đo bằng **byte**, viết kèm hậu tố thập phân `E, P, T, G, M, K` hoặc
  nhị phân `Ei, Pi, Ti, Gi, Mi, Ki` — nên `128974848`, `129e6`, `129M` và `123Mi` xấp xỉ bằng nhau.
- Mục *Nếu bạn không chỉ định memory limit*: container không có limit dùng được toàn bộ memory của
  Node và có thể kích hoạt OOM Killer; hơn nữa khi OOM xảy ra thì chính container **không có
  limit** lại là ứng viên bị kill với khả năng cao hơn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các bước cài và kiểm tra metrics-server, cùng mọi lệnh `kubectl top` | cluster lab cố ý chưa có metrics-server; Lab 3c đọc số liệu từ API object và cgroup v2 | [giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) |
| LimitRange gán memory limit mặc định cho namespace, nhắc ở mục *Nếu bạn không chỉ định memory limit* | ở đây chỉ cần biết có cơ chế đặt mặc định | bài [133](133-limit-range-vi.md) ở [giai đoạn 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) |
| Cách Node chọn nạn nhân khi bản thân Node cạn memory, và OOM Killer ở mức Node | bài chỉ nhắc hệ quả một câu; cơ chế chọn và thứ tự trục xuất là chủ đề riêng | bài [142](142-node-pressure-eviction-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |
| Nhánh *Dành cho người quản trị cluster* trong mục *Tiếp theo* (quota, ràng buộc theo namespace) | thuộc quản trị namespace, không phải cấu hình Pod | [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |

---

Trang này hướng dẫn cách gán một *request* (yêu cầu) memory và một *limit* (giới hạn) memory cho
Container. Container được đảm bảo có đủ lượng memory mà nó yêu cầu, nhưng không được phép sử
dụng nhiều memory hơn limit của nó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

Mỗi node trong cluster của bạn phải có ít nhất 300 MiB memory.

Một vài bước trên trang này yêu cầu bạn chạy dịch vụ
[metrics-server](https://github.com/kubernetes-sigs/metrics-server) trong cluster. Nếu bạn đã có
metrics-server đang chạy, bạn có thể bỏ qua các bước đó.

Nếu bạn đang chạy Minikube, hãy chạy lệnh sau để bật metrics-server:

```shell
minikube addons enable metrics-server
```

Để xem metrics-server có đang chạy hay không, hoặc một trình cung cấp khác của resource metrics
API (`metrics.k8s.io`), hãy chạy lệnh sau:

```shell
kubectl get apiservices
```

Nếu resource metrics API khả dụng, kết quả sẽ bao gồm một tham chiếu đến `metrics.k8s.io`.

```shell
NAME
v1beta1.metrics.k8s.io
```

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace mem-example
```

## Chỉ định memory request và memory limit (Specify a memory request and a memory limit)

Để chỉ định memory request cho một container, hãy thêm field `resources.requests.memory` vào
manifest tài nguyên của container. Để chỉ định memory limit, hãy thêm `resources.limits.memory`.

Trong bài thực hành này, bạn tạo một Pod có một Container. Container có memory request là
100 MiB và memory limit là 200 MiB. Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Mục `args` trong file cấu hình cung cấp các đối số cho Container khi nó khởi động. Các đối số
`"--vm-bytes", "150M"` yêu cầu Container cố gắng cấp phát 150 MiB memory.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

Xác minh rằng Container của Pod đang chạy:

```shell
kubectl get pod memory-demo --namespace=mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod memory-demo --output=yaml --namespace=mem-example
```

Kết quả cho thấy Container duy nhất trong Pod có memory request là 100 MiB và memory limit là
200 MiB.

```yaml
...
resources:
  requests:
    memory: 100Mi
  limits:
    memory: 200Mi
...
```

Chạy `kubectl top` để lấy các số liệu (metrics) của pod:

```shell
kubectl top pod memory-demo --namespace=mem-example
```

Kết quả cho thấy Pod đang sử dụng khoảng 162.900.000 byte memory, tức khoảng 150 MiB. Con số này
lớn hơn request 100 MiB của Pod, nhưng vẫn nằm trong limit 200 MiB của Pod.

```
NAME                        CPU(cores)   MEMORY(bytes)
memory-demo                 <something>  162856960
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo --namespace=mem-example
```

## Vượt quá memory limit của Container (Exceed a Container's memory limit)

Một Container có thể vượt quá memory request của nó nếu Node còn memory khả dụng. Nhưng Container
không được phép sử dụng nhiều hơn memory limit của nó. Nếu Container cấp phát nhiều memory hơn
limit, Container trở thành ứng viên bị chấm dứt (terminate). Nếu Container tiếp tục tiêu thụ
memory vượt quá limit, Container sẽ bị chấm dứt. Nếu Container bị chấm dứt có thể được khởi động
lại, kubelet sẽ khởi động lại nó, giống như với bất kỳ loại lỗi runtime nào khác.

Trong bài thực hành này, bạn tạo một Pod cố gắng cấp phát nhiều memory hơn limit của nó. Đây là
file cấu hình cho một Pod có một Container với memory request là 50 MiB và memory limit là
100 MiB:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

Trong mục `args` của file cấu hình, bạn có thể thấy Container sẽ cố gắng cấp phát 250 MiB
memory, vượt xa limit 100 MiB.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

Tại thời điểm này, Container có thể đang chạy hoặc đã bị kill. Lặp lại lệnh trên cho đến khi
Container bị kill:

```shell
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          24s
```

Xem trạng thái Container ở mức chi tiết hơn:

```shell
kubectl get pod memory-demo-2 --output=yaml --namespace=mem-example
```

Kết quả cho thấy Container đã bị kill vì hết memory (OOM):

```yaml
lastState:
   terminated:
     containerID: 65183c1877aaec2e8427bc95609cc52677a454b56fcb24340dbd22917c23b10f
     exitCode: 137
     finishedAt: 2017-06-20T20:52:19Z
     reason: OOMKilled
     startedAt: null
```

Container trong bài thực hành này có thể được khởi động lại, nên kubelet sẽ khởi động lại nó.
Lặp lại lệnh này vài lần để thấy Container liên tục bị kill rồi được khởi động lại:

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

Kết quả cho thấy Container bị kill, được khởi động lại, bị kill lần nữa, lại được khởi động lại,
và cứ thế tiếp diễn:

```
kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          37s
```
```

kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-2   1/1       Running   2          40s
```

Xem thông tin chi tiết về lịch sử của Pod:

```
kubectl describe pod memory-demo-2 --namespace=mem-example
```

Kết quả cho thấy Container khởi động rồi thất bại lặp đi lặp lại:

```
... Normal  Created   Created container with id 66a3a20aa7980e61be4922780bf9d24d1a1d8b7395c09861225b0eba1b1f8511
... Warning BackOff   Back-off restarting failed container
```

Xem thông tin chi tiết về các Node của cluster:

```
kubectl describe nodes
```

Kết quả bao gồm một bản ghi về việc Container bị kill do tình trạng hết memory (out-of-memory):

```
Warning OOMKilling Memory cgroup out of memory: Kill process 4481 (stress) score 1994 or sacrifice child
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo-2 --namespace=mem-example
```

## Chỉ định một memory request quá lớn so với các Node của bạn (Specify a memory request that is too big for your Nodes)

Memory request và limit gắn với Container, nhưng sẽ hữu ích khi coi Pod cũng có memory request
và limit. Memory request của Pod là tổng các memory request của tất cả Container trong Pod.
Tương tự, memory limit của Pod là tổng các limit của tất cả Container trong Pod.

Việc lập lịch (scheduling) Pod dựa trên request. Một Pod chỉ được lập lịch chạy trên một Node
nếu Node đó có đủ memory khả dụng để đáp ứng memory request của Pod.

Trong bài thực hành này, bạn tạo một Pod có memory request lớn đến mức vượt quá dung lượng của
bất kỳ Node nào trong cluster. Đây là file cấu hình cho một Pod có một Container với request
1000 GiB memory, nhiều khả năng vượt quá dung lượng của bất kỳ Node nào trong cluster của bạn.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-3
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-3-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "1000Gi"
      limits:
        memory: "1000Gi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
```

Xem trạng thái của Pod:

```shell
kubectl get pod memory-demo-3 --namespace=mem-example
```

Kết quả cho thấy trạng thái của Pod là PENDING. Nghĩa là Pod không được lập lịch chạy trên bất
kỳ Node nào, và nó sẽ ở trạng thái PENDING vô thời hạn:

```
kubectl get pod memory-demo-3 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-3   0/1       Pending   0          25s
```

Xem thông tin chi tiết về Pod, bao gồm các sự kiện (event):

```shell
kubectl describe pod memory-demo-3 --namespace=mem-example
```

Kết quả cho thấy Container không thể được lập lịch vì các Node không đủ memory:

```
Events:
  ...  Reason            Message
       ------            -------
  ...  FailedScheduling  No nodes are available that match all of the following predicates:: Insufficient memory (3).
```

## Đơn vị memory (Memory units)

Tài nguyên memory được đo bằng byte. Bạn có thể biểu diễn memory dưới dạng số nguyên thuần hoặc
số nguyên định điểm (fixed-point) kèm một trong các hậu tố sau: E, P, T, G, M, K, Ei, Pi, Ti,
Gi, Mi, Ki. Ví dụ, các giá trị sau biểu diễn xấp xỉ cùng một giá trị:

```
128974848, 129e6, 129M, 123Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo-3 --namespace=mem-example
```

## Nếu bạn không chỉ định memory limit (If you do not specify a memory limit)

Nếu bạn không chỉ định memory limit cho một Container, một trong các tình huống sau sẽ xảy ra:

* Container không có giới hạn trên đối với lượng memory nó sử dụng. Container có thể sử dụng
  toàn bộ memory khả dụng trên Node nơi nó đang chạy, điều này đến lượt nó có thể kích hoạt OOM
  Killer. Hơn nữa, trong trường hợp xảy ra OOM Kill, container không có giới hạn tài nguyên sẽ
  có khả năng bị kill cao hơn.

* Container đang chạy trong một namespace có memory limit mặc định, và Container được tự động
  gán limit mặc định đó. Người quản trị cluster có thể dùng
  [LimitRange](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#limitrange-v1-core)
  để chỉ định giá trị mặc định cho memory limit.

## Động lực cho memory request và limit (Motivation for memory requests and limits)

Bằng cách cấu hình memory request và limit cho các Container chạy trong cluster, bạn có thể sử
dụng hiệu quả tài nguyên memory khả dụng trên các Node của cluster. Bằng cách giữ memory request
của Pod ở mức thấp, bạn tăng cơ hội để Pod được lập lịch. Bằng cách có memory limit lớn hơn
memory request, bạn đạt được hai điều:

* Pod có thể có những đợt hoạt động bùng nổ (burst) tận dụng lượng memory đang rảnh.
* Lượng memory mà Pod có thể dùng trong một đợt bùng nổ được giới hạn ở một mức hợp lý.

## Dọn dẹp (Clean up)

Xóa namespace của bạn. Thao tác này xóa tất cả các Pod bạn đã tạo cho tác vụ này:

```shell
kubectl delete namespace mem-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên CPU cho Container và Pod (Assign CPU Resources to Containers and Pods)](./263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod (Configure Quality of Service for Pods)](288-quality-service-pod-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](289-resize-container-resources-vi.md)

### Dành cho người quản trị cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)](./232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)](./230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)](./231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)](./229-cpu-constraint-namespace-vi.md)

* [Cấu hình quota memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)](./233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình quota Pod cho một Namespace (Configure a Pod Quota for a Namespace)](./234-quota-pod-namespace-vi.md)

* [Cấu hình quota cho các đối tượng API (Configure Quotas for API Objects)](./252-quota-api-object-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](289-resize-container-resources-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 3c:

1. `lab-k8s-worker2` có 6 GB RAM và đang gần như rảnh. Bạn chạy trên đó một Pod có
   `requests.memory: "50Mi"`, `limits.memory: "100Mi"`, và tiến trình bên trong cấp phát 250M.
   Node còn thừa hàng GB — container có được mượn chỗ trống đó không? Mô tả chuỗi trạng thái bạn
   sẽ thấy khi chạy `kubectl get pod` lặp lại.
2. **Câu bẫy.** Một Pod hiện `OOMKilled`. Có phải Node đã hết memory không? Dòng nào trong
   `kubectl describe nodes` giúp bạn trả lời?
3. `Pending` kèm `Insufficient memory` và `OOMKilled` — hai thứ này khác nhau ở chỗ nào? Cái nào
   xảy ra trước khi container chạy, và cái nào xảy ra khi container đang chạy?
4. Vì sao `129M` và `123Mi` được coi là xấp xỉ cùng một giá trị? Hai hậu tố đó khác nhau ở đâu?
5. Bạn để trống memory limit cho một container "cho nhẹ manifest". Bài cảnh báo hai hệ quả gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Vượt **request** thì được mượn chỗ trống của Node, nhưng **limit là ranh giới cứng**:
   cấp phát 250M trong khi limit là 100 MiB làm container thành ứng viên bị chấm dứt, và vì nó cứ
   tiếp tục tiêu thụ nên nó **bị chấm dứt**. Container này khởi động lại được, nên kubelet khởi
   động lại nó — chạy `kubectl get pod` lặp lại sẽ thấy **`OOMKilled` → `Running` → `OOMKilled`**
   với cột `RESTARTS` tăng dần, và `describe pod` cho `BackOff restarting failed container`.
2. **Không nhất thiết.** Container bị kill vì vượt **limit của chính nó**, tức cgroup memory của nó
   cạn — Node hoàn toàn có thể còn rất nhiều memory rảnh. Bằng chứng là dòng
   **`Memory cgroup out of memory`** trong `kubectl describe nodes`: chữ **cgroup** cho biết ranh
   giới bị vượt là ranh giới của container chứ không phải của máy.
3. **`Insufficient memory` là quyết định của lập lịch**, xảy ra **trước khi** container chạy:
   scheduler so **request** với memory khả dụng của Node, không Node nào đủ nên Pod nằm `Pending`
   vô thời hạn. **`OOMKilled` xảy ra khi container đang chạy** và chạm **limit**. Một bên là bài
   toán "có chỗ đặt không", bên kia là "đang chạy có vượt trần không".
4. Vì **`M` là hậu tố thập phân còn `Mi` là hậu tố nhị phân**. Bài liệt kê thẳng
   `128974848, 129e6, 129M, 123Mi` là các cách viết xấp xỉ cùng một lượng byte — memory luôn đo
   bằng byte, phần hậu tố chỉ là cách rút gọn.
5. Một, container **không có giới hạn trên**: nó dùng được toàn bộ memory của Node và có thể kích
   hoạt **OOM Killer**. Hai — phần trái trực giác — khi OOM xảy ra thì chính container **không có
   giới hạn tài nguyên** lại **có khả năng bị kill cao hơn**. Trường hợp còn lại là namespace đã
   đặt sẵn một limit mặc định và container được gán limit đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
