# Ghi log trong Kubernetes (Logging in Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/logging/>
>
> Kiến trúc ghi log và log hệ thống.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 5/7 · Kiểm chứng ở
[Lab 11a — Observability](labs/LAB-11A-OBSERVABILITY.md) phần B7–B9 (toàn bộ trụ cột log) và
phần B1.3, nơi lab xác định trang này nằm ở nhánh nào của bài [296](296-debug-vi.md).

Đây là **trang trỏ hướng**, đọc trong một phút. Hai trang con của nó chính là hai bài bạn đã đọc
ở đầu giai đoạn 11.

**Phải hiểu ở lần đọc này:**

- Trang là cửa vào **trụ cột log** của nhánh gỡ lỗi, và trong tài liệu chính thức nó chỉ có đúng
  hai trang con: [Kiến trúc ghi log](158-logging-vi.md) và [Log hệ thống](159-system-logs-vi.md).
  Cả hai đã nằm trong danh sách đọc của giai đoạn 11.
- Dùng nó khi nào: khi cần **thu thập, truy cập và phân tích log** ở tầng cluster — đúng vai trò
  mà bài [296](296-debug-vi.md) gán cho nhánh này (quản trị viên muốn thiết lập và quản lý việc
  ghi log), chứ không phải khi cần chẩn đoán một Pod cụ thể.
- Mục thứ ba trong danh sách là một **bài viết trên blog CNCF**, tiếng Anh và nằm ngoài
  kubernetes.io — tài liệu tham khảo, không phải nguồn chuẩn để đối chiếu hành vi.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bài blog CNCF *A Practical Guide to Kubernetes Logging* | nội dung bên thứ ba, không phải tài liệu chính thức; không có gì trong lộ trình đối chiếu với nó | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo), nơi lộ trình dựng pipeline giám sát và log thật |
| Phần "các bộ công cụ ghi log (logging stack) phổ biến" mà câu dẫn của trang nhắc tới | Kubernetes không cung cấp backend log nào; đó là hệ thống ngoài cluster | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |

---

Trang này cung cấp các tài nguyên mô tả việc ghi log (logging) trong Kubernetes. Bạn có thể tìm
hiểu cách thu thập, truy cập và phân tích log bằng các công cụ tích hợp sẵn cùng các bộ công cụ
ghi log (logging stack) phổ biến:

* [Kiến trúc ghi log (Logging Architecture)](158-logging-vi.md)
* [Log hệ thống (System Logs)](159-system-logs-vi.md)
* [A Practical Guide to Kubernetes Logging](https://www.cncf.io/blog/2020/10/05/a-practical-guide-to-kubernetes-logging)
  (bài viết tiếng Anh trên blog CNCF)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trang này là cửa vào trụ cột nào, và trong tài liệu chính thức nó có đúng mấy trang con? Kể
   tên chúng.
2. **Câu bẫy.** Trang liệt kê ba mục. Mục nào trong đó **không** phải tài liệu chính thức của
   Kubernetes, và vì sao điều đó quan trọng?
3. Trên cluster lab bạn cần hai thứ: biết file log thật của một container nằm ở đâu trên
   `lab-k8s-worker2`, và biết kubelet của node đó ghi log ra đâu. Trang này chỉ bạn tới trang con
   nào cho từng nhu cầu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Cửa vào **trụ cột log**. Đúng **hai** trang con: [Kiến trúc ghi log](158-logging-vi.md) và
   [Log hệ thống](159-system-logs-vi.md). Đó cũng chính là hai bài lý thuyết bạn đã đọc ở đầu
   giai đoạn 11, nên trang này không thêm kiến thức mới — nó chỉ xác nhận ranh giới của trụ cột.
2. Mục thứ ba, **bài blog trên trang CNCF** — nó nằm ngoài kubernetes.io và viết bằng tiếng Anh.
   Quan trọng vì đây là chỗ dễ nhầm: ba mục trình bày như nhau trong cùng một danh sách, nhưng
   chỉ hai mục đầu là **nguồn chuẩn** để đối chiếu hành vi của cluster. Nội dung bên thứ ba có
   thể cũ hoặc lệch phiên bản.
3. File log của container thuộc **kiến trúc ghi log ở mức container và mức node** → trang
   [158](158-logging-vi.md). Log của kubelet là log của một **thành phần hệ thống** → trang
   [159](159-system-logs-vi.md). Trang này không tự trả lời câu nào trong hai câu đó; công dụng
   duy nhất của nó là đưa bạn tới đúng trang.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
