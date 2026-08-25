# Xử lý sự cố ứng dụng (Troubleshooting Applications)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/>
>
> Gỡ lỗi (debug) các sự cố thường gặp của ứng dụng chạy trong container.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 2/7 · Kiểm chứng ở
[Lab 11a — Observability](labs/LAB-11A-OBSERVABILITY.md) phần B7.1 và B7.3 (`kubectl logs` và
`--previous`) và phần B11 (`kubectl exec`) — đó là ba thao tác duy nhất của nhánh này dùng được
ở giai đoạn 11.

Trang này chỉ có một đoạn mô tả, không có mục nào. Nó là **trang mô tả nhánh** *Gỡ lỗi ứng dụng*
mà bài [296](296-debug-vi.md) vừa liệt kê. Đọc trong một phút, đừng tìm thao tác ở đây.

**Phải hiểu ở lần đọc này:**

- Trang khoanh vùng **ba loại chủ đề** của nhánh này: các vấn đề thường gặp với resource
  Kubernetes (Pod, Service, StatefulSet), cách hiểu **thông điệp kết thúc (termination message)**
  của container, và **các cách gỡ lỗi container đang chạy**.
- Nó chỉ **khoanh vùng**, không dạy: mọi hướng dẫn cụ thể nằm ở các trang con — bài
  [304](304-get-shell-running-container-vi.md) ngay sau đây là một trong số đó.
- Phạm vi của nhánh là **ứng dụng bạn triển khai**, không phải bản thân cluster; ranh giới này
  do bài [296](296-debug-vi.md) đặt ra và trang này nằm đúng bên trong nó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chủ đề *các vấn đề thường gặp với Pod, Service, StatefulSet* | là quy trình chẩn đoán riêng cho từng loại object, cần công cụ chưa học ở giai đoạn 11 | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), bài [299](299-debug-pods-vi.md), [301](301-debug-service-vi.md), [302](302-debug-statefulset-vi.md) |
| Chủ đề *thông điệp kết thúc (termination message) của container* | đọc được nó là kỹ năng xác định nguyên nhân Pod chết, không phải kỹ năng dựng nguồn log | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), bài [303](303-determine-reason-pod-failure-vi.md) |

---

Tài liệu này chứa một tập các tài nguyên để khắc phục sự cố với các ứng dụng chạy trong
container. Nó bao gồm những chủ đề như các vấn đề thường gặp với các resource của Kubernetes
(như Pod, Service, hoặc StatefulSet), lời khuyên về cách hiểu các thông điệp kết thúc
(termination message) của container, và các cách gỡ lỗi những container đang chạy.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trang nêu ba loại chủ đề thuộc nhánh này. Kể ra ba loại đó, và nói rõ loại nào bạn dùng được
   ngay ở giai đoạn 11.
2. **Câu bẫy.** Trang nói nó bao gồm "các cách gỡ lỗi những container đang chạy". Vậy đọc xong
   trang này bạn đã biết các cách đó chưa?
3. Một Pod bạn tự tạo trên `lab-k8s-worker2` vẫn chạy nhưng trả kết quả sai. Tình huống này rơi
   vào chủ đề nào trong ba chủ đề trang nêu, và bài nào của nhóm 11 dạy thao tác cụ thể cho nó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ba loại: **các vấn đề thường gặp với resource Kubernetes** (Pod, Service, StatefulSet);
   **cách hiểu thông điệp kết thúc (termination message) của container**; và **các cách gỡ lỗi
   những container đang chạy**. Ở giai đoạn 11 chỉ dùng được **loại thứ ba** — gỡ lỗi container
   đang chạy. Hai loại đầu là quy trình chẩn đoán, để tới
   [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố).
2. **Chưa.** Đây là chỗ dễ nhầm: trang **liệt kê chủ đề**, nó không phải hướng dẫn. Toàn bộ nội
   dung là một đoạn mô tả "tài liệu này chứa một tập các tài nguyên"; các cách gỡ lỗi thật nằm ở
   trang con, ví dụ [304](304-get-shell-running-container-vi.md). Trang mục lục cho bạn biết
   **đi đâu**, không cho bạn biết **làm gì**.
3. Rơi vào chủ đề **gỡ lỗi những container đang chạy** — container vẫn sống, vấn đề nằm bên
   trong nó, nên không phải chủ đề termination message (dành cho container đã chết) cũng không
   phải chủ đề sự cố của object. Bài dạy thao tác là
   [304 — Truy cập shell của một container đang chạy](304-get-shell-running-container-vi.md),
   thực hành ở [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) phần B11.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
