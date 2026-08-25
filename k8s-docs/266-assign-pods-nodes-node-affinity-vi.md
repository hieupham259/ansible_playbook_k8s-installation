# Gán Pod vào Node bằng Node Affinity (Assign Pods to Nodes using Node Affinity)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 7 — Lập lịch và chính sách tài nguyên](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên)
→ [7a. Scheduling và eviction](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), dòng **Thực hành**,
bài 2/3 · Kiểm chứng ở [Lab 7a — Lập lịch và eviction](labs/LAB-7A-LAP-LICH-VA-EVICTION.md), phần
B2.3–B2.4.

Bài này là bản chạy tay của mục *Node Affinity* trong bài [138](138-assign-pod-node-vi.md). Nó
đưa **hai** manifest cho **cùng một mục tiêu**, khác nhau đúng một chữ `required` và `preferred` —
đọc kỹ chỗ khác nhau đó, vì đây là điểm dễ nhầm nhất của cả nhóm.

**Phải hiểu ở lần đọc này:**

- Bước *Thêm label cho một node* là điều kiện tiên quyết cho cả hai ví dụ:
  `kubectl label nodes <your-node-name> disktype=ssd`, rồi `kubectl get nodes --show-labels` để xác
  nhận label đã lên đúng node.
- Ngữ nghĩa khác nhau của hai manifest: với `requiredDuringSchedulingIgnoredDuringExecution`, bài
  nói Pod **chỉ** được lên lịch trên node có label `disktype=ssd`; với
  `preferredDuringSchedulingIgnoredDuringExecution`, bài chỉ nói Pod **sẽ ưu tiên** node đó.
- Cấu trúc YAML của hai dạng **không giống nhau**: `required...` nhận `nodeSelectorTerms`, một
  danh sách các term chứa `matchExpressions`; `preferred...` nhận một danh sách các mục gồm
  `weight` và `preference`, và `matchExpressions` nằm bên trong `preference`.
- Điều kiện khớp viết bằng `matchExpressions` với ba trường `key`, `operator` và `values` — ở đây
  là `key: disktype`, `operator: In`, `values: [ssd]`. `values` là một **danh sách**, không phải
  một giá trị đơn.
- Bằng chứng mà bài dùng cho cả hai trường hợp là như nhau: `kubectl get pods --output=wide` rồi
  đọc cột `NODE`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hậu tố `IgnoredDuringExecution` trong tên cả hai trường | bài dùng nguyên tên mà không giải thích nửa sau nghĩa là gì | bài [138](138-assign-pod-node-vi.md#node-affinity) đã đọc ở đầu [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction); [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) B2.7 chứng minh đổi label không đuổi Pod đang chạy |
| `weight: 1` và ý nghĩa của trọng số khi có nhiều `preference` | ví dụ chỉ có một mục nên trọng số chưa tạo ra khác biệt nào quan sát được | bài [138](138-assign-pod-node-vi.md#node-affinity); [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) B2.4 |
| Câu "Kubernetes server của bạn phải ở phiên bản v1.10 hoặc mới hơn" | là mốc lịch sử của trang gốc, không phải yêu cầu của lộ trình | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) là nơi duy nhất giữ phiên bản của cluster lab |

---

Trang này hướng dẫn cách gán một Pod Kubernetes vào một node cụ thể bằng Node Affinity
trong một cluster Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.10 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

## Thêm label cho một node (Add a label to a node)

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

    ```
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,disktype=ssd,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

    Trong kết quả trên, bạn có thể thấy node `worker0` có label `disktype=ssd`.

## Lên lịch một Pod bằng node affinity bắt buộc (Schedule a Pod using required node affinity)

Manifest này mô tả một Pod có node affinity dạng
`requiredDuringSchedulingIgnoredDuringExecution` với `disktype: ssd`.
Điều này có nghĩa là Pod sẽ chỉ được lên lịch (schedule) trên node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

1. Áp dụng manifest để tạo một Pod được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx-required-affinity.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Lên lịch một Pod bằng node affinity ưu tiên (Schedule a Pod using preferred node affinity)

Manifest này mô tả một Pod có node affinity dạng
`preferredDuringSchedulingIgnoredDuringExecution` với `disktype: ssd`.
Điều này có nghĩa là Pod sẽ ưu tiên node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

1. Áp dụng manifest để tạo một Pod được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx-preferred-affinity.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Tiếp theo (What's next)

Tìm hiểu thêm về
[Node Affinity](138-assign-pod-node-vi.md#node-affinity).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7a:

1. Hai manifest của bài cùng nhắm tới label `disktype=ssd`. Chỉ ra khác biệt **cấu trúc** giữa
   `requiredDuringSchedulingIgnoredDuringExecution` và
   `preferredDuringSchedulingIgnoredDuringExecution` trong YAML — trường nào nhận cái gì.
2. **Câu bẫy.** Giả sử không node nào trong cluster mang label `disktype=ssd`, rồi bạn apply lần
   lượt hai Pod của bài. Pod nào được lập lịch, Pod nào không?
3. Bạn gắn `disktype=ssd` cho `lab-k8s-worker1`, apply Pod dùng `preferred...`, và
   `kubectl get pods --output=wide` cho thấy Pod nằm trên `lab-k8s-worker1`. Kết quả đó có **chứng
   minh** được rằng khai báo `preferred` đã có tác dụng không?
4. Trong `matchExpressions`, ba trường nào được dùng, và vì sao node affinity không cho bạn viết
   gọn thành một cặp `disktype: ssd`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `requiredDuringSchedulingIgnoredDuringExecution` nhận **`nodeSelectorTerms`** — một danh sách
   các term, mỗi term chứa `matchExpressions`. `preferredDuringSchedulingIgnoredDuringExecution`
   nhận **một danh sách các mục có `weight` và `preference`**, và `matchExpressions` nằm bên trong
   `preference`. Hai khối không hoán đổi cho nhau được dù điều kiện khớp viết y hệt.
2. **Pod `required` không được lập lịch; Pod `preferred` vẫn được lập lịch.** Bài nói Pod dùng
   `required...` "sẽ chỉ được lên lịch trên node có label `disktype=ssd`" — không có node nào như
   vậy thì không có chỗ nào hợp lệ. Còn `preferred...` bài chỉ nói Pod "sẽ ưu tiên" node có label
   đó, tức đây là nguyện vọng chứ không phải điều kiện. Hai cái tên dài gần giống nhau nên rất dễ
   đọc lướt thành một.
3. **Không đủ để chứng minh.** `preferred` chỉ nghiêng cán cân; Pod đó vẫn hoàn toàn được phép lên
   `lab-k8s-worker2`, nên việc nó rơi vào đúng node có label có thể chỉ là trùng hợp. Bài dừng ở
   mức đưa ra `kubectl get pods --output=wide` làm bằng chứng; muốn phân biệt thật thì phải tạo
   tình huống node được ưu tiên không nhận được Pod — [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md)
   B2.4 làm việc đó.
4. Ba trường là **`key`, `operator` và `values`**. Không viết gọn được vì `values` là một **danh
   sách** và `operator` quyết định cách so khớp — ở đây `In` nghĩa là giá trị của label phải nằm
   trong danh sách đó. Cú pháp này diễn đạt được nhiều hơn một cặp bằng nhau, nên nó cũng dài hơn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
