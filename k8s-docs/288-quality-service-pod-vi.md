# Cấu hình Quality of Service cho Pod (Configure Quality of Service for Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 5/9 ·
Kiểm chứng ở [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) phần B3.

Đây là vế thực hành của bài [54](54-pod-qos-vi.md). Mục tiêu duy nhất của lần đọc này: **nhìn
manifest là suy ra được QoS class**, không cần chạy Pod. Lab 3c B3 bắt bạn đoán trước rồi mới đối
chiếu với `status.qosClass`, nên đọc bài này với tâm thế học cách đoán.

**Phải hiểu ở lần đọc này:**

- `Guaranteed` đòi đủ bốn điều kiện ở mục *Tạo một Pod được gán QoS class Guaranteed*: **mọi**
  container có cả memory request lẫn memory limit, hai giá trị bằng nhau; **mọi** container có cả
  CPU request lẫn CPU limit, hai giá trị bằng nhau. Ràng buộc áp dụng như nhau cho init container;
  ephemeral container không khai được tài nguyên nên không bị tính.
- Ghi chú sau ví dụ Guaranteed: container chỉ khai **limit** mà không khai request thì Kubernetes
  **tự gán request bằng limit** — nên một manifest chỉ có `limits` vẫn có thể là `Guaranteed`.
- `Burstable` là phần còn lại có khai tài nguyên: Pod **không** đạt tiêu chí Guaranteed nhưng **ít
  nhất một** container có request hoặc limit của CPU hoặc memory. Mục *Tạo một Pod có hai Container*
  là ví dụ điển hình — một container chỉ khai `requests.memory`, container kia trống hoàn toàn.
- `BestEffort` đòi **không container nào** có bất kỳ request hay limit CPU/memory nào, tức
  `resources: {}` ở mọi container.
- QoS class được gán **lúc Pod được tạo** và **giữ nguyên suốt vòng đời** (Ghi chú đầu bài). Đọc
  giá trị thật ở `status.qosClass`, lấy nhanh bằng
  `kubectl get pod <tên> -o jsonpath='{ .status.qosClass}'` (mục *Truy xuất QoS class của một Pod*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Câu mở đầu: Kubernetes dùng QoS class để quyết định **trục xuất Pod khi tài nguyên Node bị vượt** | ở lần đọc này chỉ cần suy đúng class từ manifest; thứ tự trục xuất và ngưỡng của kubelet là chủ đề riêng | bài [142](142-node-pressure-eviction-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |
| Cơ chế `--subresource=resize` mà Ghi chú đầu bài nhắc tới khi nói control plane từ chối yêu cầu | ở đây chỉ cần nhớ **class là bất biến**; cách thực hiện một lần resize là bài riêng | bài [289](289-resize-container-resources-vi.md), bài kế tiếp trong nhóm 3c |
| Nhánh *Dành cho quản trị viên cluster* trong mục *Tiếp theo*: quota, LimitRange, Topology Manager | thuộc quản trị namespace và cấu hình kubelet ở mức node | [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace); riêng Topology Manager là bài [74](74-resource-managers-vi.md) ở [giai đoạn 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) |

---

Trang này hướng dẫn cách cấu hình các Pod sao cho chúng được gán một lớp Chất lượng dịch vụ
(Quality of Service — QoS) cụ thể. Kubernetes dùng các QoS class để đưa ra quyết định về việc
trục xuất (evict) các Pod khi tài nguyên trên Node bị vượt quá.

Khi Kubernetes tạo một Pod, nó gán cho Pod một trong các QoS class sau:

* [Guaranteed](./54-pod-qos-vi.md#guaranteed)
* [Burstable](./54-pod-qos-vi.md#burstable)
* [BestEffort](./54-pod-qos-vi.md#besteffort)

> **Ghi chú:** Kubernetes gán QoS class khi Pod được tạo, và giá trị này giữ nguyên trong suốt
> vòng đời của Pod. Nếu bạn cố
> [thay đổi kích thước tài nguyên của Pod](./289-resize-container-resources-vi.md)
> sang các giá trị dẫn tới một QoS class khác, control plane sẽ từ chối yêu cầu của bạn kèm một
> thông báo lỗi.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cũng cần có khả năng tạo và xóa các namespace.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace qos-example
```

## Tạo một Pod được gán QoS class Guaranteed (Create a Pod that gets assigned a QoS class of Guaranteed) {#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed}

Để một Pod được gán QoS class `Guaranteed`:

* Mọi Container trong Pod đều phải có memory limit và memory request.
* Với mỗi Container trong Pod, memory limit phải bằng memory request.
* Mọi Container trong Pod đều phải có CPU limit và CPU request.
* Với mỗi Container trong Pod, CPU limit phải bằng CPU request.

Các ràng buộc này áp dụng như nhau cho cả init container lẫn app container.
[Ephemeral container](./52-ephemeral-containers-vi.md) không thể định nghĩa tài nguyên nên các
ràng buộc này không áp dụng cho chúng.

Dưới đây là manifest cho một Pod có một Container. Container này có memory limit và memory
request bằng nhau, đều là 200 MiB. Container cũng có CPU limit và CPU request bằng nhau, đều là
700 milliCPU:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod.yaml --namespace=qos-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod qos-demo --namespace=qos-example --output=yaml
```

Kết quả cho thấy Kubernetes đã gán cho Pod QoS class `Guaranteed`. Kết quả cũng xác nhận rằng
Container của Pod có memory request khớp với memory limit, và có CPU request khớp với CPU
limit.

```yaml
spec:
  containers:
    ...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
    ...
status:
  qosClass: Guaranteed
```

> **Ghi chú:** Nếu một Container chỉ định memory limit của riêng nó nhưng không chỉ định memory
> request, Kubernetes sẽ tự động gán một memory request khớp với limit đó. Tương tự, nếu một
> Container chỉ định CPU limit của riêng nó nhưng không chỉ định CPU request, Kubernetes sẽ tự
> động gán một CPU request khớp với limit đó.

#### Dọn dẹp (Clean up) {#clean-up-guaranteed}

Xóa Pod của bạn:

```shell
kubectl delete pod qos-demo --namespace=qos-example
```

## Tạo một Pod được gán QoS class Burstable (Create a Pod that gets assigned a QoS class of Burstable)

Một Pod được gán QoS class `Burstable` nếu:

* Pod không đáp ứng các tiêu chí của QoS class `Guaranteed`.
* Ít nhất một Container trong Pod có memory hoặc CPU request hoặc limit.

Dưới đây là manifest cho một Pod có một Container. Container này có memory limit là 200 MiB và
memory request là 100 MiB.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-2.yaml --namespace=qos-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod qos-demo-2 --namespace=qos-example --output=yaml
```

Kết quả cho thấy Kubernetes đã gán cho Pod QoS class `Burstable`:

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: qos-demo-2-ctr
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
  ...
status:
  qosClass: Burstable
```

#### Dọn dẹp (Clean up) {#clean-up-burstable}

Xóa Pod của bạn:

```shell
kubectl delete pod qos-demo-2 --namespace=qos-example
```

## Tạo một Pod được gán QoS class BestEffort (Create a Pod that gets assigned a QoS class of BestEffort)

Để một Pod được gán QoS class `BestEffort`, các Container trong Pod không được có bất kỳ memory
hoặc CPU limit hay request nào.

Dưới đây là manifest cho một Pod có một Container. Container này không có memory hay CPU limit
hoặc request nào:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-3.yaml --namespace=qos-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod qos-demo-3 --namespace=qos-example --output=yaml
```

Kết quả cho thấy Kubernetes đã gán cho Pod QoS class `BestEffort`:

```yaml
spec:
  containers:
    ...
    resources: {}
  ...
status:
  qosClass: BestEffort
```

#### Dọn dẹp (Clean up) {#clean-up-besteffort}

Xóa Pod của bạn:

```shell
kubectl delete pod qos-demo-3 --namespace=qos-example
```

## Tạo một Pod có hai Container (Create a Pod that has two Containers)

Dưới đây là manifest cho một Pod có hai Container. Một Container chỉ định memory request là
200 MiB. Container còn lại không chỉ định bất kỳ request hay limit nào.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-4
  namespace: qos-example
spec:
  containers:

  - name: qos-demo-4-ctr-1
    image: nginx
    resources:
      requests:
        memory: "200Mi"

  - name: qos-demo-4-ctr-2
    image: redis
```

Lưu ý rằng Pod này đáp ứng các tiêu chí của QoS class `Burstable`. Nghĩa là, nó không đáp ứng
các tiêu chí của QoS class `Guaranteed`, và một trong các Container của nó có memory request.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-4.yaml --namespace=qos-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod qos-demo-4 --namespace=qos-example --output=yaml
```

Kết quả cho thấy Kubernetes đã gán cho Pod QoS class `Burstable`:

```yaml
spec:
  containers:
    ...
    name: qos-demo-4-ctr-1
    resources:
      requests:
        memory: 200Mi
    ...
    name: qos-demo-4-ctr-2
    resources: {}
    ...
status:
  qosClass: Burstable
```

## Truy xuất QoS class của một Pod (Retrieve the QoS class for a Pod)

Thay vì xem tất cả các trường, bạn có thể chỉ xem đúng trường mình cần:

```bash
kubectl --namespace=qos-example get pod qos-demo-4 -o jsonpath='{ .status.qosClass}{"\n"}'
```

```none
Burstable
```

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace qos-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](./264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](./263-assign-cpu-resource-vi.md)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](./232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](./230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](./231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](./229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](./233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình Quota Pod cho một Namespace](./234-quota-pod-namespace-vi.md)

* [Cấu hình Quota cho các API Object](./252-quota-api-object-vi.md)

* [Điều khiển các chính sách Topology Management trên một node](./259-topology-manager-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 3c:

1. Suy QoS class của ba Pod sau, chỉ bằng cách nhìn manifest: (a) một container, chỉ có
   `limits: {cpu: 500m, memory: 128Mi}`, không có dòng `requests` nào; (b) hai container, container
   đầu chỉ có `requests.memory: 200Mi`, container sau không khai gì; (c) một container với
   `resources: {}`.
2. **Câu bẫy.** Một Pod có duy nhất một container, khai `requests.memory: 200Mi` và
   `limits.memory: 200Mi` — bằng nhau chằn chặn — nhưng không khai CPU. Nó là `Guaranteed` chứ?
3. Trên cluster lab, bạn tạo một Pod `Burstable` rồi `kubectl edit` nó cho request bằng limit ở cả
   CPU lẫn memory. Pod có trở thành `Guaranteed` không? Trường nào là nơi bạn đọc câu trả lời thật,
   và lệnh nào lấy đúng trường đó mà không phải in cả YAML?
4. Init container có bị tính vào tiêu chí của `Guaranteed` không? Còn ephemeral container?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. (a) **`Guaranteed`.** Chỉ khai limit thì Kubernetes **tự gán request bằng limit**, nên container
   có đủ cả bốn giá trị và request bằng limit ở cả hai loại tài nguyên. (b) **`Burstable`** — không
   đạt Guaranteed, nhưng có ít nhất một container khai request memory; đây đúng là ví dụ *Tạo một
   Pod có hai Container* trong bài. (c) **`BestEffort`** — không container nào khai bất kỳ request
   hay limit CPU/memory nào.
2. **Không, nó là `Burstable`.** Chỗ bẫy là ta chỉ nhìn "request bằng limit" mà quên rằng
   `Guaranteed` đòi **cả bốn** điều kiện: phải có **cả CPU lẫn memory**, mỗi loại đều có cả request
   lẫn limit và bằng nhau. Thiếu hẳn CPU thì tiêu chí Guaranteed không đạt, và vì vẫn có khai memory
   nên Pod rơi vào `Burstable`.
3. **Không.** Ghi chú đầu bài nói rõ: QoS class được gán **khi Pod được tạo** và **giữ nguyên trong
   suốt vòng đời của Pod**; nếu bạn đổi tài nguyên sang các giá trị dẫn tới class khác thì **control
   plane từ chối yêu cầu** kèm thông báo lỗi. Trường cần đọc là **`status.qosClass`**, lấy bằng
   `kubectl get pod <tên> -o jsonpath='{ .status.qosClass}'`.
4. Init container **có bị tính**: bài nói các ràng buộc của `Guaranteed` áp dụng như nhau cho cả
   init container lẫn app container. Ephemeral container thì **không**, vì chúng **không thể định
   nghĩa tài nguyên**, nên các ràng buộc đó không áp dụng cho chúng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
