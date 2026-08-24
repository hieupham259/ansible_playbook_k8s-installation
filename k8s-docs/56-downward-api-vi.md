# Downward API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/downward-api/>
>
> Có hai cách để phơi bày (expose) các trường của Pod và container cho một container đang chạy:
> biến môi trường, và dưới dạng các file được điền bởi một loại volume đặc biệt.
> Hai cách phơi bày các trường của Pod và container này được gọi chung là downward API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 10/11 · Kiểm chứng
ở Lab 3a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài chủ yếu là **ba danh sách trường**. Đừng học thuộc chúng; thứ phải nhớ là **ranh giới giữa ba
danh sách đó** — trường nào dùng được cả hai cơ chế, trường nào chỉ có ở biến môi trường, trường
nào chỉ có ở volume. Nhầm ranh giới này là lỗi manifest thường gặp.

**Phải hiểu ở lần đọc này:**

- Mục đích: cho container biết thông tin về chính nó **mà không cần Kubernetes client hay API
  server**, để giữ gắn kết lỏng với Kubernetes.
- Hai cơ chế phơi bày và chỉ hai: **biến môi trường**, và **file trong một volume `downwardAPI`**.
  Chỉ một số trường của Kubernetes API là khả dụng qua chúng.
- Ranh giới theo cấp: **`fieldRef`** lấy các trường **cấp Pod**, **`resourceFieldRef`** lấy các
  trường **cấp container** (request và limit).
- Ba nhóm trường của `fieldRef`: dùng được **cả hai cơ chế** (`metadata.name`,
  `metadata.namespace`, `metadata.uid`, `metadata.annotations['<KEY>']`,
  `metadata.labels['<KEY>']`); **chỉ biến môi trường** (`spec.serviceAccountName`,
  `spec.nodeName`, `status.hostIP`, `status.hostIPs`, `status.podIP`, `status.podIPs`); **chỉ
  volume** (`metadata.labels` và `metadata.annotations` ở dạng đầy đủ, mỗi mục một dòng).
- Hành vi dự phòng: nếu container không đặt limit CPU/bộ nhớ mà bạn vẫn phơi bày chúng, kubelet
  trả về **giá trị allocatable tối đa của node**, chứ không phải rỗng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `spec.serviceAccountName` | chưa học ServiceAccount | giai đoạn 9 — bài [118](118-service-accounts-vi.md) |
| Volume `downwardAPI` với tư cách một loại volume | chưa học volume | giai đoạn 6 — bài [91](91-volumes-vi.md), [93](93-projected-volumes-vi.md) |
| Ghi chú về resize: volume được cập nhật còn biến môi trường thì không | resize dựa trên requests/limits chưa học | giai đoạn 3, nhóm 3b — bài [110](110-manage-resources-containers-vi.md) |
| `hugepages-*` và `ephemeral-storage` trong `resourceFieldRef` | là tài nguyên chưa học | giai đoạn 3, nhóm 3b — bài [110](110-manage-resources-containers-vi.md); lưu trữ tạm ở giai đoạn 6 — bài [95](95-ephemeral-storage-vi.md) |
| Phép tính *node allocatable* trong phần dự phòng | thuộc phần tài nguyên của node | giai đoạn 7 — bài [142](142-node-pressure-eviction-vi.md) |

---

Đôi khi việc một container có thông tin về chính nó là hữu ích, mà không cần gắn kết
(coupled) quá chặt với Kubernetes. _Downward API_ cho phép các container sử dụng thông
tin về chính chúng hoặc về cluster mà không cần dùng Kubernetes client hay API server.

Một ví dụ là một ứng dụng có sẵn giả định rằng một biến môi trường quen thuộc nào đó chứa
một định danh (identifier) duy nhất. Một khả năng là bọc (wrap) ứng dụng lại, nhưng cách
đó tẻ nhạt, dễ gây lỗi, và vi phạm mục tiêu gắn kết lỏng (low coupling). Một lựa chọn tốt
hơn là dùng tên của Pod làm định danh, và tiêm (inject) tên của Pod vào biến môi trường
quen thuộc đó.

Trong Kubernetes, có hai cách để phơi bày các trường của Pod và container cho một
container đang chạy:

* dưới dạng [biến môi trường](336-env-variable-expose-pod-info-vi.md)
* dưới dạng [các file trong một volume `downwardAPI`](335-downward-api-volume-vi.md)

Hai cách phơi bày các trường của Pod và container này được gọi chung là _downward API_.

## Các trường khả dụng (Available fields) {#available-fields}

Chỉ một số trường của Kubernetes API là khả dụng thông qua downward API. Mục này liệt kê
những trường bạn có thể cung cấp.

Bạn có thể truyền thông tin từ các trường khả dụng ở cấp Pod bằng `fieldRef`. Ở cấp độ
API, `spec` của một Pod luôn định nghĩa ít nhất một
[Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container).
Bạn có thể truyền thông tin từ các trường khả dụng ở cấp Container bằng
`resourceFieldRef`.

### Thông tin khả dụng qua `fieldRef` (Information available via `fieldRef`) {#downwardapi-fieldRef}

Với một số trường cấp Pod, bạn có thể cung cấp chúng cho container dưới dạng biến môi
trường hoặc dùng một volume `downwardAPI`. Các trường khả dụng qua cả hai cơ chế gồm:

`metadata.name`
: tên của Pod

`metadata.namespace`
: namespace của Pod

`metadata.uid`
: ID duy nhất của Pod

`metadata.annotations['<KEY>']`
: giá trị của annotation có tên `<KEY>` của Pod (ví dụ: `metadata.annotations['myannotation']`)

`metadata.labels['<KEY>']`
: giá trị dạng chữ của label có tên `<KEY>` của Pod (ví dụ: `metadata.labels['mylabel']`)

Các thông tin sau khả dụng qua biến môi trường **nhưng không khả dụng dưới dạng fieldRef
của volume downwardAPI**:

`spec.serviceAccountName`
: tên service account của Pod

`spec.nodeName`
: tên của node nơi Pod đang thực thi

`status.hostIP`
: địa chỉ IP chính của node mà Pod được gán vào

`status.hostIPs`
: các địa chỉ IP này là phiên bản dual-stack của `status.hostIP`, địa chỉ đầu tiên luôn giống với `status.hostIP`.

`status.podIP`
: địa chỉ IP chính của Pod (thường là địa chỉ IPv4 của nó)

`status.podIPs`
: các địa chỉ IP này là phiên bản dual-stack của `status.podIP`, địa chỉ đầu tiên luôn giống với `status.podIP`

Các thông tin sau khả dụng qua `fieldRef` của volume `downwardAPI`, **nhưng không khả
dụng dưới dạng biến môi trường**:

`metadata.labels`
: tất cả các label của Pod, được định dạng `label-key="escaped-label-value"` với mỗi label trên một dòng

`metadata.annotations`
: tất cả các annotation của Pod, được định dạng `annotation-key="escaped-annotation-value"` với mỗi annotation trên một dòng

### Thông tin khả dụng qua `resourceFieldRef` (Information available via `resourceFieldRef`) {#downwardapi-resourceFieldRef}

Các trường cấp container này cho phép bạn cung cấp thông tin về
[request và limit](110-manage-resources-containers-vi.md#requests-and-limits)
của các tài nguyên như CPU và bộ nhớ.

> **Ghi chú:**
>
> **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`
>
> Tài nguyên CPU và bộ nhớ của container có thể được thay đổi kích thước (resize) trong
> khi container đang chạy. Nếu điều này xảy ra, volume downward API sẽ được cập nhật,
> nhưng các biến môi trường sẽ không được cập nhật trừ khi container khởi động lại.
> Xem [Thay đổi kích thước tài nguyên CPU và bộ nhớ đã gán cho Container](289-resize-container-resources-vi.md)
> để biết thêm chi tiết.

`resource: limits.cpu`
: Limit CPU của container

`resource: requests.cpu`
: Request CPU của container

`resource: limits.memory`
: Limit bộ nhớ của container

`resource: requests.memory`
: Request bộ nhớ của container

`resource: limits.hugepages-*`
: Limit hugepages của container

`resource: requests.hugepages-*`
: Request hugepages của container

`resource: limits.ephemeral-storage`
: Limit ephemeral-storage (lưu trữ tạm thời) của container

`resource: requests.ephemeral-storage`
: Request ephemeral-storage của container

#### Thông tin dự phòng cho limit tài nguyên (Fallback information for resource limits)

Nếu limit CPU và bộ nhớ không được chỉ định cho một container, và bạn dùng downward API
để cố phơi bày thông tin đó, thì kubelet mặc định sẽ phơi bày giá trị có thể cấp phát
(allocatable) tối đa cho CPU và bộ nhớ dựa trên phép tính
[node allocatable](253-reserve-compute-resources-vi.md#node-allocatable).

## Tiếp theo (What's next)

Bạn có thể đọc về [volume `downwardAPI`](91-volumes-vi.md#downwardapi).

Bạn có thể thử dùng downward API để phơi bày thông tin cấp container hoặc cấp Pod:
* dưới dạng [biến môi trường](336-env-variable-expose-pod-info-vi.md)
* dưới dạng [các file trong volume `downwardAPI`](335-downward-api-volume-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Bạn muốn container nhận **tất cả** label của Pod. Dùng biến môi trường được không?
2. Ứng dụng cần biết nó đang chạy trên `k8s-worker1` hay `k8s-worker2`. Trường nào cho biết điều
   đó, và nó dùng được qua cơ chế nào?
3. Vì sao Kubernetes làm downward API, thay vì để container tự gọi API server hỏi về chính nó?
4. Container không đặt `limits.memory`, nhưng manifest vẫn phơi bày `resource: limits.memory`.
   Container nhận được giá trị gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** `metadata.labels` ở dạng **đầy đủ** — tất cả label, định dạng
   `label-key="escaped-label-value"` mỗi label một dòng — **chỉ khả dụng qua `fieldRef` của volume
   `downwardAPI`, không khả dụng dưới dạng biến môi trường**. Qua biến môi trường bạn chỉ lấy được
   **từng label một** bằng `metadata.labels['<KEY>']`. `metadata.annotations` cũng chia đôi y hệt.
   Đây là chỗ dễ nhầm vì hai cơ chế trông như thay thế được cho nhau, nhưng ba nhóm trường của
   chúng chỉ giao nhau một phần.
2. Trường **`spec.nodeName`** — tên của node nơi Pod đang thực thi. Nó nằm trong nhóm **chỉ khả
   dụng qua biến môi trường**, không lấy được bằng `fieldRef` của volume `downwardAPI`. Cùng nhóm
   đó còn có `status.hostIP` và `status.hostIPs` nếu bạn cần địa chỉ IP của node thay vì tên.
3. Để **gắn kết lỏng**: downward API cho container dùng thông tin về chính nó **mà không cần
   Kubernetes client hay API server**. Bài lấy ví dụ một ứng dụng có sẵn giả định một biến môi
   trường quen thuộc chứa định danh duy nhất — cách bọc ứng dụng lại thì **tẻ nhạt, dễ gây lỗi và
   vi phạm chính mục tiêu gắn kết lỏng**, còn cách tốt hơn là tiêm thẳng tên Pod vào biến đó.
4. **Giá trị allocatable tối đa của node cho bộ nhớ.** Khi limit CPU và bộ nhớ không được chỉ định
   cho container mà bạn vẫn phơi bày chúng, **kubelet mặc định phơi bày giá trị có thể cấp phát
   tối đa** dựa trên phép tính node allocatable. Hệ quả thực tế: ứng dụng tự chỉnh kích thước
   heap theo `limits.memory` sẽ tưởng nó được dùng gần hết node.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
