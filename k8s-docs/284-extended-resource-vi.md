# Gán Extended Resource cho một Container (Assign Extended Resources to a Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 4/9 ·
Kiểm chứng ở [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) phần B5.

Bài rất ngắn và chỉ là **một nửa câu chuyện**: nửa *tiêu thụ*. Nửa *quảng bá* — làm cho Node công
bố ra bốn dongle — nằm ở bài [209](209-extended-resource-node-vi.md), đọc ở giai đoạn 25. Vì vậy
mục *Trước khi bạn bắt đầu* yêu cầu làm bài 209 trước; Lab 3c B5.1 lấp chỗ đó bằng đúng lệnh PATCH
đã trình bày trong bài [110](110-manage-resources-containers-vi.md).

**Phải hiểu ở lần đọc này:**

- Cách yêu cầu, ở mục *Gán một extended resource cho một Pod*: thêm
  `resources.requests.<resource_name>` (và `limits` tương ứng) vào manifest của **container**. Tên
  hợp lệ có dạng `example.com/foo` — phần trước dấu `/` là tên miền của tổ chức bạn.
- Điều kiện tiên quyết, ở mục *Trước khi bạn bắt đầu*: phải có một Node **đang quảng bá** tài
  nguyên dongle thì Pod mới xin được. Không có Node nào công bố thì không có gì để cấp phát.
- Mục *Thử tạo một Pod thứ hai*: extended resource được **đếm và trừ dần**. Pod đầu chiếm 3 trong
  4 dongle, nên Pod thứ hai xin 2 dongle không tìm được Node nào đáp ứng.
- Hệ quả của việc đó, cũng ở mục đó: Pod thứ hai **vẫn được tạo** nhưng ở trạng thái `Pending`,
  với condition `PodScheduled: False` và event
  `FailedScheduling ... Insufficient example.com/dongle`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bài [209](209-extended-resource-node-vi.md) mà mục *Trước khi bạn bắt đầu* bắt làm trước | đó là vế quản trị Node, không phải cấu hình Pod; Lab 3c B5.1 dùng lệnh PATCH của bài [110](110-manage-resources-containers-vi.md) để thay thế | [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |
| Nhánh *Dành cho quản trị viên cluster* trong mục *Tiếp theo* | cùng lý do trên: thuộc phần quản trị Node và namespace | [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách gán các tài nguyên mở rộng (extended resource) cho một Container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

Trước khi làm bài thực hành này, hãy làm bài thực hành trong
[Quảng bá Extended Resource cho một Node](./209-extended-resource-node-vi.md).
Bài đó sẽ cấu hình một trong các Node của bạn để quảng bá tài nguyên dongle.

## Gán một extended resource cho một Pod (Assign an extended resource to a Pod)

Để yêu cầu một extended resource, hãy đưa trường `resources.requests.<resource_name>` vào
manifest của container. `*.kubernetes.io/`. Tên extended resource hợp lệ có dạng
`example.com/foo`, trong đó `example.com` được thay bằng tên miền (domain) của tổ chức bạn
và `foo` là một tên tài nguyên mang tính mô tả.

Đây là file cấu hình cho một Pod có một Container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo
spec:
  containers:
  - name: extended-resource-demo-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 3
      limits:
        example.com/dongle: 3
```

Trong file cấu hình, bạn có thể thấy Container yêu cầu 3 dongle.

Tạo một Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod.yaml
```

Xác minh rằng Pod đang chạy:

```shell
kubectl get pod extended-resource-demo
```

Mô tả (describe) Pod:

```shell
kubectl describe pod extended-resource-demo
```

Đầu ra cho thấy các yêu cầu về dongle:

```yaml
Limits:
  example.com/dongle: 3
Requests:
  example.com/dongle: 3
```

## Thử tạo một Pod thứ hai (Attempt to create a second Pod)

Đây là file cấu hình cho một Pod có một Container. Container này yêu cầu hai dongle.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo-2
spec:
  containers:
  - name: extended-resource-demo-2-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 2
      limits:
        example.com/dongle: 2
```

Kubernetes sẽ không thể đáp ứng yêu cầu hai dongle, bởi vì Pod đầu tiên đã dùng ba trong
bốn dongle khả dụng.

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod-2.yaml
```

Mô tả Pod:

```shell
kubectl describe pod extended-resource-demo-2
```

Đầu ra cho thấy Pod không thể được lập lịch (schedule), bởi vì không có Node nào còn 2
dongle khả dụng:

```
Conditions:
  Type    Status
  PodScheduled  False
...
Events:
  ...
  ... Warning   FailedScheduling  pod (extended-resource-demo-2) failed to fit in any node
fit failure summary on nodes : Insufficient example.com/dongle (1)
```

Xem trạng thái của Pod:

```shell
kubectl get pod extended-resource-demo-2
```

Đầu ra cho thấy Pod đã được tạo, nhưng không được lập lịch để chạy trên một Node.
Nó có trạng thái Pending:

```yaml
NAME                       READY     STATUS    RESTARTS   AGE
extended-resource-demo-2   0/1       Pending   0          6m
```

## Dọn dẹp (Clean up)

Xóa các Pod mà bạn đã tạo cho bài thực hành này:

```shell
kubectl delete pod extended-resource-demo
kubectl delete pod extended-resource-demo-2
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For application developers)

* [Gán tài nguyên bộ nhớ cho Container và Pod](./264-assign-memory-resource-vi.md)
* [Gán tài nguyên CPU cho Container và Pod](./263-assign-cpu-resource-vi.md)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Quảng bá Extended Resource cho một Node](./209-extended-resource-node-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 3c:

1. Lab 3c quảng bá **4 dongle trên `lab-k8s-worker2`**, còn `lab-k8s-worker1` không quảng bá gì.
   Pod thứ nhất xin 3 dongle và chạy được. Pod thứ hai xin 2 dongle. Chuyện gì xảy ra, và
   `lab-k8s-worker1` có cứu được Pod thứ hai không?
2. **Câu bẫy.** Pod thứ hai hiện `Pending`. Vậy object Pod đó đã tồn tại trong cluster hay lệnh
   `kubectl apply` đã bị từ chối? Bạn nhìn vào đâu để biết chắc?
3. Vì sao tên tài nguyên phải viết là `example.com/dongle` chứ không phải `dongle`?
4. Bạn bỏ qua mục *Trước khi bạn bắt đầu* và apply thẳng Pod xin 3 dongle trên một cluster chưa
   Node nào quảng bá dongle. Kết quả là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod thứ hai **`Pending`**. Bốn dongle đều nằm trên `lab-k8s-worker2` và Pod đầu đã tiêu 3, chỉ
   còn 1 — không đủ 2. `lab-k8s-worker1` **không cứu được**: nó không quảng bá dongle nào, nên với
   scheduler nó có 0 đơn vị `example.com/dongle`. Event ghi
   **`Insufficient example.com/dongle`**.
2. **Object đã được tạo.** Bài nói rõ: "Pod đã được tạo, nhưng không được lập lịch để chạy trên một
   Node". Đây là chỗ dễ nhầm — thất bại khi lập lịch **không** phải là thất bại khi tạo object.
   Bằng chứng: `kubectl get pod` vẫn liệt kê nó với `STATUS Pending`, và `kubectl describe` cho
   condition **`PodScheduled: False`** kèm event `FailedScheduling` — muốn có condition thì object
   phải tồn tại đã.
3. Vì bài quy định **tên extended resource hợp lệ có dạng `example.com/foo`**: phần trước dấu `/`
   là **tên miền của tổ chức bạn**, phần sau là tên mô tả tài nguyên. Đặt tên trần như `dongle` là
   không theo khuôn đó.
4. **Không có gì để cấp phát**, nên Pod không thể được lập lịch và nằm `Pending` với
   `Insufficient example.com/dongle` — đúng như tình huống Pod thứ hai trong bài. Chính vì thế mục
   *Trước khi bạn bắt đầu* bắt phải cấu hình một Node quảng bá dongle trước.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
