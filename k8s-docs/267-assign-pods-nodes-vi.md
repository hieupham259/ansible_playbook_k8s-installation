# Gán Pod vào Node (Assign Pods to Nodes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 7 — Lập lịch và chính sách tài nguyên](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên)
→ [7a. Scheduling và eviction](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), dòng **Thực hành**,
bài 3/3 · Kiểm chứng ở [Lab 7a — Lập lịch và eviction](labs/LAB-7A-LAP-LICH-VA-EVICTION.md), phần
B2.1–B2.2.

Đây là **bài cuối của nhóm Thực hành 7a**. Nó bắt đầu bằng đúng bước gắn label như bài
[266](266-assign-pods-nodes-node-affinity-vi.md) vừa đọc, rồi rẽ sang hai cách đơn giản nhất để
ghim Pod: `nodeSelector` và `nodeName`. Đọc để chốt lại ranh giới giữa ba cú pháp bạn vừa gặp.

**Phải hiểu ở lần đọc này:**

- Mục *Thêm label cho một node*: label nằm trên **object Node**, gắn bằng
  `kubectl label nodes <your-node-name> disktype=ssd` và đọc bằng `kubectl get nodes --show-labels`.
  Output ví dụ cho thấy mỗi node vốn đã mang sẵn label `kubernetes.io/hostname`.
- Mục *Tạo một Pod được lên lịch vào node bạn đã chọn*: `nodeSelector` là một cặp `key: value`
  **phẳng** đặt trong `spec` của Pod — bài viết `nodeSelector: disktype: ssd` — và Pod sẽ được lên
  lịch trên node có label khớp.
- Mục *Tạo một Pod được lên lịch vào một node cụ thể*: `nodeName` ghi thẳng **tên node** vào Pod
  spec. Đây là con đường khác hẳn: nó không cần label nào, và cũng không mô tả điều kiện nào.
- Bằng chứng của bài: `kubectl get pods --output=wide` rồi đọc cột `NODE`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hệ quả của việc `nodeName` bỏ qua scheduler | trang này chỉ đưa manifest, không nói gì về hệ quả | bài [137](137-kube-scheduler-vi.md) và [138](138-assign-pod-node-vi.md#nodename) đã đọc ở đầu [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction); [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) B1.3 và B4.5 |
| Các cách ràng buộc khác — node affinity, taint và toleration, topology spread | trang này cố ý chỉ trình bày hai cách đơn giản nhất | bài [266](266-assign-pods-nodes-node-affinity-vi.md) vừa đọc và bài [139](139-taint-and-toleration-vi.md), [140](140-topology-spread-constraints-vi.md) ở đầu [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction); [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) B2–B5 |
| Mục *Tiếp theo* — label và selector nói chung | ở đây chỉ cần biết label gắn trên Node | bài [18](18-labels-vi.md) đã đọc ở [1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) |

---

Trang này hướng dẫn cách gán một Pod Kubernetes vào một node cụ thể trong một cluster
Kubernetes.

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

## Thêm label cho một node (Add a label to a node) {#add-a-label-to-a-node}

1. Liệt kê các node trong cluster của bạn, kèm theo các label của chúng:

    ```shell
    kubectl get nodes --show-labels
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

1. Chọn một trong các node của bạn và thêm label cho nó:

    ```shell
    kubectl label nodes <your-node-name> disktype=ssd
    ```

    trong đó `<your-node-name>` là tên của node bạn đã chọn.

1. Xác nhận rằng node bạn chọn đã có label `disktype=ssd`:

    ```shell
    kubectl get nodes --show-labels
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,disktype=ssd,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

    Trong kết quả trên, bạn có thể thấy node `worker0` có label `disktype=ssd`.

## Tạo một Pod được lên lịch vào node bạn đã chọn (Create a pod that gets scheduled to your chosen node)

File cấu hình Pod này mô tả một Pod có node selector `disktype: ssd`. Điều này có nghĩa là
Pod sẽ được lên lịch (schedule) trên một node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

1. Dùng file cấu hình để tạo một Pod sẽ được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Tạo một Pod được lên lịch vào một node cụ thể (Create a pod that gets scheduled to specific node)

Bạn cũng có thể lên lịch một Pod vào đúng một node cụ thể bằng cách đặt `nodeName`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node # lên lịch Pod vào một node cụ thể
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

Dùng file cấu hình để tạo một Pod sẽ chỉ được lên lịch vào node `foo-node`.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [label và selector](18-labels-vi.md).
* Tìm hiểu thêm về [node](23-nodes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7a:

1. `nodeSelector` và `nodeName` cùng đưa Pod về một node xác định. Chúng khác nhau ở chỗ nào —
   mỗi cái khớp bằng thứ gì?
2. **Câu bẫy.** So `nodeSelector` của bài này với node affinity `required...` của bài
   [266](266-assign-pods-nodes-node-affinity-vi.md): cả hai đều là ràng buộc cứng và cùng nhắm
   `disktype=ssd`. Cú pháp khác nhau ra sao, và cái nào diễn đạt được "`disktype` là `ssd` **hoặc**
   `nvme`"?
3. Trên cluster lab, `lab-k8s-master` là control plane **có taint**. Bạn gắn `disktype=ssd` cho
   chính `lab-k8s-master` rồi apply Pod `nginx` có `nodeSelector: disktype: ssd`. Bài nói Pod "sẽ
   được lên lịch trên một node có label `disktype=ssd`". Vậy Pod có chạy trên `lab-k8s-master`
   không?
4. Bạn muốn ghim một Pod vào đúng `lab-k8s-worker2` mà **không** gắn thêm label mới cho node nào.
   Output `kubectl get nodes --show-labels` trong bài gợi ý cho bạn hai cách — đó là hai cách nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `nodeSelector` khớp qua **label của node**: bạn mô tả cặp `key: value` mà node phải có, và node
   nào mang label đó đều là ứng viên hợp lệ. `nodeName` khớp qua **tên node**: bạn ghi thẳng tên
   vào Pod spec, không cần node mang label nào. Bài đặt hai mục riêng đúng theo khác biệt đó — một
   mục là "node bạn đã chọn", mục kia là "một node cụ thể".
2. `nodeSelector` là **cặp `key: value` phẳng**, khớp bằng nhau và chỉ diễn đạt được một giá trị.
   Node affinity dùng **`matchExpressions`** với `key`, `operator` và `values` là một **danh sách**,
   nên `operator: In` cùng `values: [ssd, nvme]` diễn đạt được yêu cầu "một trong hai". Đây là chỗ
   dễ nhầm: hai cú pháp cho cùng một kết quả trong ví dụ đơn giản, nhưng không tương đương về sức
   diễn đạt.
3. **Không.** `nodeSelector` chỉ nói node nào **đủ điều kiện được chọn** — nó là bộ lọc, không phải
   mệnh lệnh, nên nó không gỡ được thứ đang đẩy Pod ra khỏi node. Taint trên control plane vẫn
   chặn Pod như bài [139](139-taint-and-toleration-vi.md) đã trình bày. Gắn thêm label chỉ mở rộng
   tập ứng viên, không bỏ qua được các điều kiện khác.
4. Hai cách: dùng **`nodeName: lab-k8s-worker2`**; hoặc dùng **`nodeSelector` với label có sẵn**
   `kubernetes.io/hostname: lab-k8s-worker2` — output ví dụ trong bài cho thấy mọi node đều đã mang
   sẵn label `kubernetes.io/hostname` trước khi bạn gắn thêm gì.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Trả lời trôi chảy được cả ba bài thực
hành của nhóm rồi thì mở [Lab 7a — Lập lịch và eviction](labs/LAB-7A-LAP-LICH-VA-EVICTION.md): B2.1
và B2.2 chạy lại chính bài này, B2.3 và B2.4 chạy bài
[266](266-assign-pods-nodes-node-affinity-vi.md), còn B6.1 kiểm chứng bài
[210](210-guaranteed-scheduling-critical-addon-pods-vi.md) trên add-on thật của cluster.
