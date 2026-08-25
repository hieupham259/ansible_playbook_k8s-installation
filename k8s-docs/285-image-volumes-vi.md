# Sử dụng Image Volume với một Pod (Use an Image Volume With a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 6/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B12.

Bài ngắn và chỉ có hai ví dụ. Điều đáng nhớ không nằm ở YAML mà ở chỗ: đây là loại volume **phụ
thuộc container runtime**, không phải thứ API server một mình quyết định được.

**Phải hiểu ở lần đọc này:**

- `volumes[*].image.reference` mount **nội dung của một object trong OCI registry** vào container
  qua `volumeMounts` — hoàn toàn **tách khỏi image mà container đang chạy**. Trong ví dụ,
  container chạy `debian` nhưng `/volume` lại là nội dung của `quay.io/crio/artifact:v2`.
- `pullPolicy` nằm **bên trong `image` của volume**, tức volume có chính sách kéo image riêng,
  không dùng chung `imagePullPolicy` của container.
- Ba điều kiện ở mục *Trước khi bạn bắt đầu* mà bài liệt kê riêng: **container runtime phải hỗ
  trợ tính năng image volume**, bạn phải chạy được lệnh trên host, và exec được vào Pod. Điều
  kiện đầu là điểm mấu chốt — API server chấp nhận manifest chưa có nghĩa là Pod chạy được.
- `subPath` (và `subPathExpr`) dùng được với image volume từ v1.33: `subPath: dir` khiến container
  chỉ thấy phần `dir` của volume, nên `/volume/file` ở ví dụ hai chính là `/volume/dir/file` của
  ví dụ một — đầu ra đổi từ `2` sang `1`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* gợi ý minikube và các playground | lộ trình không dùng minikube; cluster lab đã dựng sẵn | [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Hai link `subPath` / `subPathExpr` sang bài Volume | Volume là một chủ đề riêng, chưa học | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |
| Mục *Đọc thêm* — "Volume kiểu `image`" | như trên | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách cấu hình một pod sử dụng image volume. Tính năng này cho phép bạn
mount nội dung từ các OCI registry vào bên trong container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.31 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

- Container runtime cần hỗ trợ tính năng image volume
- Bạn cần có khả năng thực thi (exec) lệnh trên host
- Bạn cần có khả năng exec vào trong các pod

## Chạy một Pod sử dụng image volume (Run a Pod that uses an image volume) {#create-pod}

Image volume cho một pod được bật bằng cách đặt trường `volumes[*].image` của `.spec` thành
một tham chiếu (reference) hợp lệ và sử dụng nó trong `volumeMounts` của container. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

1. Tạo pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/image-volumes.yaml
   ```

1. Attach vào container:

   ```shell
   kubectl exec image-volume -it -- bash
   ```

1. Kiểm tra nội dung của một file trong volume:

   ```shell
   cat /volume/dir/file
   ```

   Đầu ra tương tự như:

   ```none
   1
   ```

   Bạn cũng có thể kiểm tra một file khác ở một đường dẫn khác:

   ```shell
   cat /volume/file
   ```

   Đầu ra tương tự như:

   ```none
   2
   ```

## Sử dụng `subPath` (hoặc `subPathExpr`) (Use `subPath` (or `subPathExpr`))

Từ Kubernetes v1.33, bạn có thể sử dụng
[`subPath`](./91-volumes-vi.md#using-subpath) hoặc
[`subPathExpr`](./91-volumes-vi.md#using-subpath-expanded-environment)
khi dùng tính năng image volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
      subPath: dir
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

1. Tạo pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/image-volumes-subpath.yaml
   ```

1. Attach vào container:

   ```shell
   kubectl exec image-volume -it -- bash
   ```

1. Kiểm tra nội dung của file từ sub path `dir` trong volume:

   ```shell
   cat /volume/file
   ```

   Đầu ra tương tự như:

   ```none
   1
   ```

## Đọc thêm (Further reading)

- [Volume kiểu `image`](./91-volumes-vi.md#image)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trong manifest ví dụ có **hai** tham chiếu image ở hai chỗ khác nhau. Chỉ ra hai chỗ đó và nói
   rõ vai trò từng chỗ.
2. **Câu bẫy.** Bạn apply manifest, API server nhận, `kubectl get pod` hiện Pod đã tạo. Đã đủ để
   kết luận image volume hoạt động chưa? Mục *Trước khi bạn bắt đầu* đòi thêm điều kiện gì?
3. Ví dụ đầu `cat /volume/file` ra `2`, ví dụ sau cũng `cat /volume/file` nhưng ra `1`. Manifest
   khác nhau đúng một dòng — dòng nào, và vì sao kết quả đổi?
4. Pod `image-volume` chạy trên `lab-k8s-worker2` của cluster lab. Bạn muốn chứng minh nội dung
   trong `/volume` thật sự đến từ image kia chứ không phải từ image `debian` của container. Theo
   bài, làm thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỗ thứ nhất là **`containers[].image: debian`** — image mà container thực sự chạy, tức
   filesystem gốc và tiến trình `sleep infinity`. Chỗ thứ hai là
   **`volumes[].image.reference: quay.io/crio/artifact:v2`** — object trong OCI registry mà
   Kubernetes mount **thành một volume** vào container tại `mountPath: /volume`. Hai thứ độc lập
   nhau: cái đầu là môi trường chạy, cái sau là dữ liệu được đưa vào.
2. **Chưa đủ.** Bài tách riêng một điều kiện tiên quyết: **container runtime cần hỗ trợ tính năng
   image volume** (kèm hai điều kiện thao tác: chạy được lệnh trên host, exec được vào pod). Đây
   là chỗ trực giác dễ sai: quen với việc "API server nhận manifest là xong", trong khi image
   volume là **tính năng phải được hiện thực ở tầng runtime trên node** — API server nhận trường
   `volumes[].image` không nói gì về việc node có mount được nó hay không.
3. Dòng thêm vào là **`subPath: dir`** trong `volumeMounts` của container. Nó khiến container chỉ
   thấy **phần `dir` của volume** thay vì toàn bộ volume, nên đường dẫn `/volume/file` của ví dụ
   hai thực chất là `/volume/dir/file` của ví dụ một — mà file đó chứa `1`. Bài ghi rõ `subPath`
   và `subPathExpr` dùng được với image volume **từ Kubernetes v1.33**.
4. Exec vào container rồi đọc các file mà **chỉ image kia mới có**: `kubectl exec image-volume -it
   -- bash`, rồi `cat /volume/dir/file` (ra `1`) và `cat /volume/file` (ra `2`). Hai file này
   thuộc nội dung của `quay.io/crio/artifact:v2`, không phải của `debian` — chúng xuất hiện dưới
   `/volume` chứng tỏ volume đã được mount từ registry.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
