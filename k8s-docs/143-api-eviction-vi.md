# Eviction khởi phát qua API (API-initiated Eviction)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 8/13 ·
Kiểm chứng ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md).

Bài ngắn nhưng **quan trọng với vận hành**: đây là cơ chế đứng sau `kubectl drain`, lệnh bạn
sẽ dùng mỗi lần bảo trì hay nâng cấp một node. Đọc nó ngay sau bài
[142](142-node-pressure-eviction-vi.md) để thấy rõ hai cơ chế cùng tên "eviction" nhưng đối
lập nhau ở gần như mọi điểm.

Đừng bận tâm phần JSON và `curl`; trong thực tế bạn gần như luôn gọi qua `kubectl drain`.

**Phải hiểu ở lần đọc này:**

- Eviction qua API là việc **tạo một object `Eviction`** cho một Pod; API server nhận và chấm
  dứt Pod. Bài mô tả nó là "một thao tác `DELETE` **có kiểm soát theo chính sách**".
- Nó **tôn trọng** PodDisruptionBudget và `terminationGracePeriodSeconds` — ngược hẳn với
  eviction do áp lực node.
- Ba phản hồi và ý nghĩa: `200 OK` cho phép; `429 Too Many Requests` là PDB đang chặn (hoặc
  rate limit của API); `500 Internal Server Error` là cấu hình sai, ví dụ nhiều PDB cùng tham
  chiếu một Pod. Pod không thuộc workload nào có PDB thì **luôn** nhận `200 OK`.
- Trình tự sáu bước sau khi được cho phép: API server đánh dấu deletion timestamp và grace
  period → kubelet bắt đầu tắt Pod êm thấm → **control plane gỡ Pod khỏi EndpointSlice trong
  lúc kubelet còn đang tắt** → hết grace period kubelet cưỡng bức chấm dứt → kubelet báo API
  server → API server xóa object Pod.
- *Xử lý sự cố eviction bị kẹt*: khi Pod thay thế không đạt `Ready`, Eviction API trả `429`
  hoặc `500` mãi. Lối thoát là dừng thao tác tự động và điều tra, hoặc xóa thẳng Pod thay vì
  dùng Eviction API.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai khối JSON `policy/v1` và `policy/v1beta1`, ví dụ `curl` | chỉ cần khi gọi API trực tiếp; `kubectl drain` đã gói sẵn | không cần |
| Cách cấu hình PodDisruptionBudget | lý thuyết đã đọc ở bài [53](53-disruptions-vi.md); phần thao tác nằm ở nhánh tasks | [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) |

---

Eviction khởi phát qua API (API-initiated eviction) là quá trình bạn sử dụng
[Eviction API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#create-eviction-pod-v1-core)
để tạo một đối tượng `Eviction` nhằm kích hoạt việc chấm dứt Pod một cách êm thấm (graceful pod termination).

Bạn có thể yêu cầu eviction bằng cách gọi trực tiếp Eviction API, hoặc theo cách
lập trình bằng một client của API server, chẳng hạn lệnh `kubectl drain`. Thao tác này
tạo ra một đối tượng `Eviction`, khiến API server chấm dứt (terminate) Pod.

Các eviction khởi phát qua API tôn trọng các
[`PodDisruptionBudgets`](339-configure-pdb-vi.md)
và [`terminationGracePeriodSeconds`](47-pod-lifecycle-vi.md#pod-termination)
mà bạn đã cấu hình.

Việc dùng API để tạo một đối tượng Eviction cho một Pod giống như thực hiện một
[thao tác `DELETE`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#delete-delete-a-pod)
có kiểm soát theo chính sách (policy-controlled) trên Pod đó.

## Gọi Eviction API (Calling the Eviction API)

Bạn có thể dùng một [client theo ngôn ngữ lập trình của Kubernetes](190-access-cluster-api-vi.md#programmatic-access-to-the-api)
để truy cập Kubernetes API và tạo một đối tượng `Eviction`. Để làm điều đó, bạn
POST thao tác muốn thực hiện, tương tự ví dụ sau:

#### policy/v1

> **Ghi chú:** Eviction `policy/v1` khả dụng từ v1.22 trở đi. Dùng `policy/v1beta1` với các bản phát hành trước đó.

```json
{
  "apiVersion": "policy/v1",
  "kind": "Eviction",
  "metadata": {
    "name": "quux",
    "namespace": "default"
  }
}
```

#### policy/v1beta1

> **Ghi chú:** Đã ngưng dùng (deprecated) từ v1.22, được thay thế bởi `policy/v1`.

```json
{
  "apiVersion": "policy/v1beta1",
  "kind": "Eviction",
  "metadata": {
    "name": "quux",
    "namespace": "default"
  }
}
```

Ngoài ra, bạn có thể thử một thao tác eviction bằng cách truy cập API qua
`curl` hoặc `wget`, tương tự ví dụ sau:

```bash
curl -v -H 'Content-type: application/json' https://your-cluster-api-endpoint.example/api/v1/namespaces/default/pods/quux/eviction -d @eviction.json
```

## Cách hoạt động của eviction khởi phát qua API (How API-initiated eviction works)

Khi bạn yêu cầu một eviction qua API, API server thực hiện các kiểm tra
admission và phản hồi theo một trong các cách sau:

* `200 OK`: eviction được cho phép, subresource `Eviction` được tạo, và
  Pod bị xóa, tương tự như việc gửi một request `DELETE` đến URL của Pod.
* `429 Too Many Requests`: eviction hiện không được phép do
  PodDisruptionBudget đã cấu hình.
  Bạn có thể thử lại eviction sau đó. Bạn cũng có thể thấy
  phản hồi này do cơ chế giới hạn tốc độ (rate limiting) của API.
* `500 Internal Server Error`: eviction không được phép vì có
  cấu hình sai, ví dụ như khi nhiều PodDisruptionBudget cùng tham chiếu đến một Pod.

Nếu Pod mà bạn muốn evict không thuộc một workload có
PodDisruptionBudget, API server luôn trả về `200 OK` và cho phép
eviction.

Nếu API server cho phép eviction, Pod bị xóa theo trình tự sau:

1. Tài nguyên `Pod` trong API server được cập nhật với một dấu thời gian xóa
   (deletion timestamp), sau thời điểm đó API server coi tài nguyên `Pod` là đã chấm dứt.
   Tài nguyên `Pod` cũng được đánh dấu với grace period (thời gian ân hạn) đã cấu hình.
1. kubelet trên node nơi Pod cục bộ đang chạy nhận thấy tài nguyên `Pod`
   được đánh dấu để chấm dứt và bắt đầu tắt Pod cục bộ
   một cách êm thấm (gracefully).
1. Trong khi kubelet đang tắt Pod, control plane gỡ Pod
   khỏi các đối tượng EndpointSlice.
   Kết quả là các controller không còn coi Pod là một đối tượng hợp lệ nữa.
1. Sau khi grace period của Pod hết hạn, kubelet cưỡng bức chấm dứt
   (forcefully terminate) Pod cục bộ.
1. kubelet báo cho API server gỡ bỏ tài nguyên `Pod`.
1. API server xóa tài nguyên `Pod`.

## Xử lý sự cố eviction bị kẹt (Troubleshooting stuck evictions)

Trong một số trường hợp, các ứng dụng của bạn có thể rơi vào trạng thái hỏng,
khi đó Eviction API sẽ chỉ trả về các phản hồi `429` hoặc `500` cho đến khi bạn
can thiệp. Điều này có thể xảy ra nếu, ví dụ, một ReplicaSet tạo các pod cho
ứng dụng của bạn nhưng các pod mới không đạt được trạng thái `Ready`. Bạn cũng có thể
nhận thấy hành vi này trong các trường hợp Pod bị evict gần nhất có termination grace period dài.

Nếu bạn nhận thấy các eviction bị kẹt, hãy thử một trong các giải pháp sau:

* Hủy bỏ hoặc tạm dừng thao tác tự động đang gây ra sự cố. Điều tra
  ứng dụng bị kẹt trước khi bạn khởi động lại thao tác đó.
* Chờ một lúc, sau đó xóa trực tiếp Pod khỏi control plane của cluster
  thay vì dùng Eviction API.

## Tiếp theo (What's next)

* Tìm hiểu cách bảo vệ ứng dụng của bạn với một [Pod Disruption Budget](339-configure-pdb-vi.md).
* Tìm hiểu về [Eviction do áp lực node (Node-pressure Eviction)](142-node-pressure-eviction-vi.md).
* Tìm hiểu về [Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)](141-pod-priority-preemption-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Bạn chạy `kubectl drain lab-k8s-worker2`. Thực chất `kubectl` tạo ra object gì, và thành phần
   nào chấm dứt Pod?
2. Cùng gọi là "eviction", nhưng eviction qua API và eviction do áp lực node ở bài
   [142](142-node-pressure-eviction-vi.md) đối xử với PodDisruptionBudget và
   `terminationGracePeriodSeconds` khác nhau thế nào?
3. Một Deployment chỉ có 1 replica và được bảo vệ bằng PDB `minAvailable: 1`. Bạn drain node
   đang chạy Pod đó. Bạn nhận mã trả về nào, và vì sao lệnh không tự thoát ra được?
4. Một Pod trên `lab-k8s-worker2` đang đứng sau một Service. Trong sáu bước xóa Pod, ở bước nào Pod
   ngừng nhận request mới — trước hay sau khi container thật sự chết?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `kubectl drain` là một **client của API server**; nó **tạo một object `Eviction`** cho từng
   Pod trên node, và chính **API server** chấm dứt Pod. Bạn không xóa Pod trực tiếp: bài mô tả
   việc này giống một thao tác `DELETE` **có kiểm soát theo chính sách** trên Pod.
2. **Ngược nhau hoàn toàn.** Eviction qua API **tôn trọng** cả PodDisruptionBudget lẫn
   `terminationGracePeriodSeconds` mà bạn đã cấu hình — đó chính là phần "kiểm soát theo chính
   sách". Eviction do áp lực node thì kubelet làm một mình và **không tôn trọng** cả hai. Cùng
   một chữ "eviction" nhưng một bên đi qua tầng API và chịu mọi ràng buộc ở đó, một bên là
   phản xạ tự vệ của node.
3. **`429 Too Many Requests`**, lặp mãi. Với PDB `minAvailable: 1` trên workload chỉ có 1
   replica, không bao giờ có thời điểm nào evict mà không vi phạm budget, nên API server luôn
   từ chối. Đây đúng là tình huống *eviction bị kẹt*: Eviction API sẽ chỉ trả `429` (hoặc `500`)
   cho đến khi bạn can thiệp. Lối thoát: hủy hoặc tạm dừng thao tác tự động rồi sửa PDB / tăng
   số replica, hoặc chờ rồi **xóa thẳng Pod** khỏi control plane thay vì dùng Eviction API.
   Lưu ý `500` mang nghĩa khác hẳn: cấu hình sai, ví dụ nhiều PDB cùng trỏ vào một Pod.
4. **Bước 3, và nó xảy ra trước khi container chết.** Trình tự là: API server đặt deletion
   timestamp và grace period (bước 1) → kubelet bắt đầu tắt Pod êm thấm (bước 2) → **trong khi
   kubelet đang tắt Pod, control plane gỡ Pod khỏi các đối tượng EndpointSlice** (bước 3), nên
   các controller không còn coi Pod là endpoint hợp lệ. Chỉ tới bước 4, khi grace period hết
   hạn, kubelet mới cưỡng bức chấm dứt Pod. Nhờ thứ tự này mà Pod có khoảng thời gian xử lý nốt
   request đang dở trong lúc đã ngừng nhận request mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
