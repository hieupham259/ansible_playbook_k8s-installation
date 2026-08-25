# Giám sát trong Kubernetes (Monitoring in Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/monitoring/>
>
> Giám sát các thành phần hệ thống của Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 6/7 · Kiểm chứng ở
[Lab 11a — Observability](labs/LAB-11A-OBSERVABILITY.md) phần B2 (endpoint `/metrics`) và phần
B10 (cấu hình tracing), cộng phần B1.3 nơi lab xác định trang này nằm ở nhánh nào của bài
[296](296-debug-vi.md).

Đây là **trang trỏ hướng**, đọc trong một phút và là cặp song sinh của bài
[316](316-debug-logging-vi.md): cùng khuôn, khác trụ cột.

**Phải hiểu ở lần đọc này:**

- Trang là cửa vào **trụ cột giám sát**, và nó có đúng hai trang con:
  [Metric cho các thành phần hệ thống](160-system-metrics-vi.md) và
  [Trace cho các thành phần hệ thống](161-system-traces-vi.md) — cả hai đã nằm trong danh sách
  đọc của giai đoạn 11.
- Ranh giới quan trọng nằm ngay ở câu dẫn: cả hai trang con nói về **thành phần hệ thống của
  Kubernetes**, không phải về ứng dụng bạn triển khai.
- Ranh giới với bài [316](316-debug-logging-vi.md): trang này không nhắc gì tới log. Metric và
  trace ở đây, log ở bên kia — ba trụ cột được chia thành hai trang trỏ hướng chứ không một.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Việc "thu thập metric" theo nghĩa dựng một hệ thống thu thập và lưu trữ | trang chỉ dẫn tới nguồn metric; bản thân Kubernetes không kèm hệ thống thu thập nào | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo), nơi lộ trình dựng stack giám sát thật |
| Việc thu thập trace theo nghĩa có backend nhận span | phải bật tracing trên kube-apiserver và kubelet rồi thêm collector, tức sửa cấu hình control plane | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy); ở giai đoạn 11 chỉ **đọc** cấu hình, xem [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) phần B10 |

---

Trang này cung cấp các tài nguyên mô tả việc giám sát (monitoring) trong Kubernetes. Bạn có thể
tìm hiểu cách thu thập metric hệ thống và trace cho các thành phần hệ thống của Kubernetes:

* [Metric cho các thành phần hệ thống Kubernetes (Metrics For Kubernetes System Components)](160-system-metrics-vi.md)
* [Trace cho các thành phần hệ thống Kubernetes (Traces For Kubernetes System Components)](161-system-traces-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trang này là cửa vào trụ cột nào, có mấy trang con, và vì sao trụ cột log **không** nằm ở đây?
2. **Câu bẫy.** Trang tên là "Giám sát trong Kubernetes". Vậy nó có chỉ bạn cách giám sát ứng
   dụng mà bạn triển khai lên cluster không?
3. Bạn muốn biết kube-apiserver trên `lab-k8s-master` đang phục vụ bao nhiêu request. Trang này
   đưa bạn tới trang con nào, và vì sao không phải trang con kia?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Cửa vào **trụ cột giám sát**, gồm đúng **hai** trang con:
   [160 — Metric cho các thành phần hệ thống](160-system-metrics-vi.md) và
   [161 — Trace cho các thành phần hệ thống](161-system-traces-vi.md). Trụ cột log **không** nằm
   ở đây vì nó có trang trỏ hướng riêng — bài [316](316-debug-logging-vi.md). Ba trụ cột được
   chia làm hai cửa: metric và trace một bên, log một bên.
2. **Không.** Đây là chỗ dễ nhầm vì tên trang rất rộng. Câu dẫn của trang nói rõ nó về việc thu
   thập **metric hệ thống và trace cho các thành phần hệ thống của Kubernetes**, và cả hai trang
   con đều mang đúng chữ đó trong tiêu đề. Giám sát ứng dụng của bạn là việc khác và không thuộc
   trang này.
3. Tới [160 — Metric cho các thành phần hệ thống](160-system-metrics-vi.md), vì kube-apiserver là
   một **thành phần hệ thống** và số request là một **metric** nó phát ra. Không phải
   [161](161-system-traces-vi.md) vì trace trả lời câu hỏi khác — một request đã đi qua những
   chặng nào và mỗi chặng tốn bao lâu — chứ không đếm số lượng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
