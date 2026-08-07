# Mức sẵn sàng lập lịch của Pod (Pod Scheduling Readiness)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](LO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 10/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Mọi bài trước trong nhóm đều trả lời câu hỏi "Pod này lên node nào". Bài này trả lời một câu
khác: "**khi nào** thì được phép bắt đầu hỏi câu đó". Nó là một công tắc chặn ở ngay đầu hàng
đợi, do bên ngoài bật tắt.

Ví dụ trong bài chạy được nguyên văn trên cluster lab — `test-pod` chỉ dùng image `pause` và
không xin tài nguyên gì, nên đây là một trong số ít bài của nhóm bạn thử được ngay mà không
cần dựng tình huống.

**Phải hiểu ở lần đọc này:**

- `.spec.schedulingGates` là danh sách chuỗi; còn ít nhất một gate thì Pod ở trạng thái
  **`SchedulingGated`** và scheduler **chưa xét tới nó**.
- Trường này **chỉ được khởi tạo lúc tạo Pod** (bởi client hoặc do admission mutate). Sau khi
  tạo, các gate có thể được **gỡ theo thứ tự tùy ý**, nhưng **không được thêm gate mới**.
- Gỡ **hết** gate thì Pod mới vào hàng đợi lập lịch bình thường. Kiểm chứng bằng
  `kubectl get pod` (cột STATUS) và `kubectl get pod <tên> -o jsonpath='{.spec.schedulingGates}'`
  — kết quả rỗng là đã sạch gate.
- Lý do tính năng tồn tại: Pod "thiếu tài nguyên thiết yếu" nằm chờ lâu gây **xáo trộn (churn)
  không cần thiết** cho scheduler và cho các thành phần hạ nguồn như Cluster AutoScaler. Gate
  tách bạch "chưa sẵn sàng để xét" khỏi "đã xét và không lập lịch được".
- Trong lúc Pod còn gate, bạn chỉ được **siết chặt** các chỉ thị lập lịch — thêm
  `nodeSelector`, thêm biểu thức vào node affinity `required` — chứ không được nới lỏng hay
  sửa cái đã có; riêng phần `preferredDuringSchedulingIgnoredDuringExecution` thì sửa tùy ý.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Khả năng quan sát* — `scheduler_pending_pods{queue="gated"}` | cần stack thu thập metric | giai đoạn 11, bài [160](160-system-metrics-vi.md) |
| Bốn quy tắc chi tiết trong *Các chỉ thị lập lịch có thể thay đổi của Pod* và lý giải OR/AND | chỉ cần khi tự viết controller đặt và gỡ gate | giai đoạn 14, bài [181](181-operator-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.30 [stable]`

Trước đây, các Pod được coi là sẵn sàng để lập lịch ngay khi được tạo. Kubernetes scheduler
luôn cần mẫn tìm các node để đặt tất cả các Pod đang pending. Tuy nhiên, trong
thực tế, một số Pod có thể ở trạng thái "thiếu tài nguyên thiết yếu" (miss-essential-resources) trong một thời gian dài.
Những Pod này thực chất gây xáo trộn (churn) cho scheduler (và cả các thành phần tích hợp hạ nguồn như Cluster AutoScaler)
một cách không cần thiết.

Bằng cách chỉ định/gỡ bỏ trường `.spec.schedulingGates` của một Pod, bạn có thể kiểm soát
thời điểm một Pod sẵn sàng được xem xét để lập lịch.

## Cấu hình schedulingGates của Pod (Configuring Pod schedulingGates)

Trường `schedulingGates` chứa một danh sách các chuỗi, và mỗi chuỗi được hiểu là một
tiêu chí mà Pod phải thỏa mãn trước khi được coi là có thể lập lịch. Trường này chỉ có thể
được khởi tạo khi Pod được tạo (bởi client, hoặc được biến đổi (mutate) trong quá trình admission).
Sau khi tạo, mỗi schedulingGate có thể được gỡ bỏ theo thứ tự tùy ý, nhưng không được phép thêm scheduling gate mới.

![pod-scheduling-gates-diagram](https://kubernetes.io/docs/images/podSchedulingGates.svg)

*Hình. SchedulingGates của Pod ([xem sơ đồ](https://mermaid.live/edit#pako:eNplkktTwyAUhf8KgzuHWpukaYszutGlK3caFxQuCVMCGSDVTKf_XfKyPlhxz4HDB9wT5lYAptgHFuBRsdKxenFMClMYFIdfUdRYgbiD6ItJTEbR8wpEq5UpUfnDTf-5cbPoJjcbXdcaE61RVJIiqJvQ_Y30D-OCt-t3tFjcR5wZayiVnIGmkv4NiEfX9jijKTmmRH5jf0sRugOP0HyHUc1m6KGMFP27cM28fwSJDluPpNKaXqVJzmFNfHD2APRKSjnNFx9KhIpmzSfhVls3eHdTRrwG8QnxKfEZUUNeYTDBNbiaKRF_5dSfX-BQQQ0FpnEqQLJWhwIX5hyXsjbYl85wTINrgeC2EZd_xFQy7b_VJ6GCdd-itkxALE84dE3fAqXyIUZya6Qqe711OspVCI2ny2Vv35QqVO3-htt66ZWomAvVcZcv8yTfsiSFfJOydZoKvl_ttjLJVlJsblcJw-czwQ0zr9ZeqGDgeR77b2jD8xdtjtDn))*

## Ví dụ sử dụng (Usage example)

Để đánh dấu một Pod là chưa sẵn sàng để lập lịch, bạn có thể tạo nó với một hoặc nhiều scheduling gate như sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  schedulingGates:
  - name: example.com/foo
  - name: example.com/bar
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.6
```

Sau khi Pod được tạo, bạn có thể kiểm tra trạng thái của nó bằng:

```bash
kubectl get pod test-pod
```

Kết quả cho thấy Pod đang ở trạng thái `SchedulingGated`:

```none
NAME       READY   STATUS            RESTARTS   AGE
test-pod   0/1     SchedulingGated   0          7s
```

Bạn cũng có thể kiểm tra trường `schedulingGates` của nó bằng cách chạy:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.schedulingGates}'
```

Kết quả là:

```none
[{"name":"example.com/foo"},{"name":"example.com/bar"}]
```

Để báo cho scheduler biết rằng Pod này đã sẵn sàng để lập lịch, bạn có thể gỡ bỏ hoàn toàn
`schedulingGates` của nó bằng cách áp dụng lại một manifest đã chỉnh sửa:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.6
```

Bạn có thể kiểm tra xem `schedulingGates` đã được xóa hết chưa bằng cách chạy:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.schedulingGates}'
```

Kết quả được kỳ vọng là rỗng. Và bạn có thể kiểm tra trạng thái mới nhất của Pod bằng cách chạy:

```bash
kubectl get pod test-pod -o wide
```

Vì test-pod không yêu cầu bất kỳ tài nguyên CPU/bộ nhớ nào, trạng thái của Pod này được kỳ vọng
chuyển từ `SchedulingGated` trước đó sang `Running`:

```none
NAME       READY   STATUS    RESTARTS   AGE   IP         NODE
test-pod   1/1     Running   0          15s   10.0.0.4   node-2
```

## Khả năng quan sát (Observability)

Metric `scheduler_pending_pods` đi kèm một nhãn mới `"gated"` để phân biệt giữa một Pod
đã được thử lập lịch nhưng bị kết luận là không thể lập lịch (unschedulable), với một Pod được
đánh dấu tường minh là chưa sẵn sàng để lập lịch. Bạn có thể dùng
`scheduler_pending_pods{queue="gated"}` để xem kết quả của metric này.

## Các chỉ thị lập lịch có thể thay đổi của Pod (Mutable Pod scheduling directives)

Bạn có thể thay đổi các chỉ thị lập lịch (scheduling directive) của Pod trong khi chúng vẫn còn scheduling gate, với một số ràng buộc nhất định.
Ở mức tổng quát, bạn chỉ có thể siết chặt các chỉ thị lập lịch của một Pod. Nói cách khác, các chỉ thị
sau khi cập nhật sẽ khiến các Pod chỉ có thể được lập lịch trên một tập con của những node
mà trước đó nó có thể khớp. Cụ thể hơn, các quy tắc cập nhật chỉ thị lập lịch của một Pod như sau:

1. Với `.spec.nodeSelector`, chỉ cho phép thêm mới. Nếu trường này chưa có, sẽ được phép đặt nó.

2. Với `spec.affinity.nodeAffinity`, nếu đang là nil, thì được phép đặt bất kỳ giá trị nào.

3. Nếu `NodeSelectorTerms` đang rỗng, sẽ được phép đặt.
   Nếu không rỗng, thì chỉ cho phép thêm các `NodeSelectorRequirements` vào `matchExpressions`
   hoặc `fieldExpressions`, và không cho phép bất kỳ thay đổi nào đối với các `matchExpressions`
   và `fieldExpressions` hiện có. Lý do là các term trong
   `.requiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms` được OR với nhau,
   trong khi các biểu thức trong `nodeSelectorTerms[].matchExpressions` và
   `nodeSelectorTerms[].fieldExpressions` được AND với nhau.

4. Với `.preferredDuringSchedulingIgnoredDuringExecution`, mọi cập nhật đều được cho phép.
   Lý do là các term ở mức preferred không mang tính ràng buộc bắt buộc (authoritative), nên các policy controller
   không kiểm định (validate) các term này.

## Tiếp theo (What's next)

* Đọc [KEP PodSchedulingReadiness](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/3521-pod-scheduling-readiness) để biết thêm chi tiết

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Một Pod `SchedulingGated` và một Pod `Pending` vì không node nào đủ tài nguyên — cả hai đều
   chưa chạy. Khác nhau ở chỗ nào từ góc nhìn của scheduler?
2. Bạn đã tạo một Pod không có gate và giờ muốn giữ nó lại bằng cách thêm `schedulingGates`.
   Làm được không? Vì sao?
3. Trên cluster lab, bạn tạo `test-pod` theo đúng manifest của bài với hai gate
   `example.com/foo` và `example.com/bar`, rồi apply lại một manifest chỉ bỏ gate
   `example.com/foo`. Pod chạy chưa? Cần làm gì và kiểm chứng bằng lệnh nào?
4. Vì sao Kubernetes thêm cơ chế này thay vì cứ để Pod nằm `Pending` như trước?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod `SchedulingGated` là Pod **được đánh dấu tường minh là chưa sẵn sàng để lập lịch** —
   scheduler **chưa hề thử** đặt nó. Pod `Pending` vì thiếu tài nguyên là Pod **đã được thử
   lập lịch nhưng bị kết luận là không thể lập lịch (unschedulable)**. Đây đúng là hai giá trị
   mà metric `scheduler_pending_pods` phải tách ra bằng nhãn `gated`. Nhầm hai trạng thái này
   dẫn tới chẩn đoán sai: một bên phải sửa tài nguyên/ràng buộc, một bên phải đi tìm ai chưa gỡ
   gate.
2. **Không.** Trường `schedulingGates` **chỉ có thể được khởi tạo khi Pod được tạo** — bởi
   client hoặc do bị mutate trong quá trình admission. Sau khi tạo, **không được phép thêm
   scheduling gate mới**; bạn chỉ được gỡ. Muốn chặn thì phải chặn từ lúc tạo Pod.
3. **Chưa chạy.** Mỗi gate có thể gỡ theo thứ tự tùy ý, nhưng Pod chỉ được xem xét lập lịch khi
   **không còn gate nào** — ở đây `example.com/bar` vẫn còn. Cần **gỡ bỏ hoàn toàn**
   `schedulingGates` rồi apply lại. Kiểm chứng:
   `kubectl get pod test-pod -o jsonpath='{.spec.schedulingGates}'` phải trả về **rỗng**, và
   `kubectl get pod test-pod` chuyển từ `SchedulingGated` sang `Running` — `test-pod` không xin
   tài nguyên CPU/bộ nhớ nào nên nó lên được ngay.
4. Vì các Pod ở trạng thái "thiếu tài nguyên thiết yếu" trong thời gian dài **gây xáo trộn
   (churn) không cần thiết cho scheduler** và cho các thành phần tích hợp hạ nguồn như Cluster
   AutoScaler: scheduler cứ thử đi thử lại một Pod mà bên ngoài biết chắc là chưa thể chạy.
   `schedulingGates` chuyển quyền quyết định thời điểm sang cho bên đang giữ điều kiện đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
