# Gỡ lỗi một StatefulSet (Debug a StatefulSet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-statefulset/>
>
> Tác vụ này chỉ cho bạn cách gỡ lỗi một StatefulSet.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối, giai đoạn 24 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố),
bài 10/10 · Các trang CP không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là trang rất ngắn, gần như một bảng chỉ đường: StatefulSet **không có quy trình debug
riêng** — bài chỉ gồm một bước liệt kê Pod theo label rồi rẽ sang các tài liệu chuyên biệt
tùy triệu chứng. Lý thuyết về StatefulSet bạn đã đọc ở [bài 65](65-statefulset-vi.md).

**Phải hiểu ở lần đọc này:**

- Điểm vào duy nhất là **liệt kê Pod của StatefulSet bằng label selector**
  (`kubectl get pods -l app.kubernetes.io/name=MyApp`) — giống mọi workload khác; bạn cần
  biết label mà template của StatefulSet gắn lên Pod, không phải tên StatefulSet.
- **Hai lối rẽ tùy triệu chứng**: Pod kẹt ở `Unknown` hoặc `Terminating` **kéo dài** thì sang
  tác vụ *Deleting StatefulSet Pods* (xóa Pod của StatefulSet có quy trình riêng); còn lỗi
  bên trong từng Pod cụ thể thì dùng đúng quy trình *Debug Pods* chung.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung chi tiết của tác vụ *Deleting StatefulSet Pods* được trỏ tới | trang này chỉ đặt link; quy trình xóa cưỡng bức Pod của StatefulSet có rủi ro riêng, đọc khi gặp thật | mở link gốc trên kubernetes.io khi gặp Pod kẹt `Unknown`/`Terminating` — trang đó không nằm trong danh sách giai đoạn 24 |

---

Tác vụ này chỉ cho bạn cách gỡ lỗi một StatefulSet.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn.
* Bạn nên có sẵn một StatefulSet đang chạy mà bạn muốn điều tra.

## Gỡ lỗi một StatefulSet (Debugging a StatefulSet)

Để liệt kê tất cả các Pod thuộc về một StatefulSet, trong đó các Pod được gắn label
`app.kubernetes.io/name=MyApp`, bạn có thể dùng lệnh sau:

```shell
kubectl get pods -l app.kubernetes.io/name=MyApp
```

Nếu bạn thấy bất kỳ Pod nào trong danh sách ở trạng thái `Unknown` hoặc `Terminating` trong
một khoảng thời gian dài, hãy tham khảo tác vụ
[Xóa các Pod của StatefulSet](340-delete-stateful-set-vi.md)
để biết hướng dẫn xử lý chúng.
Bạn có thể gỡ lỗi từng Pod riêng lẻ trong một StatefulSet bằng hướng dẫn
[Gỡ lỗi Pod](299-debug-pods-vi.md).

## Tiếp theo (What's next)

Tìm hiểu thêm về
[gỡ lỗi init container](298-debug-init-containers-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint giai đoạn 24:

1. Một Pod của StatefulSet trên `k8s-worker2` của bạn kẹt ở `Terminating` rất lâu sau khi
   node đó mất liên lạc. Theo trang này, bạn mở tài liệu nào tiếp theo — *Debug Pods* hay
   *Deleting StatefulSet Pods*?
2. Bạn muốn xem vì sao một Pod cụ thể của StatefulSet đang crash liên tục. Trang này có đưa
   ra quy trình debug riêng cho StatefulSet không, hay bạn dùng quy trình nào?
3. Lệnh nào liệt kê mọi Pod thuộc một StatefulSet, và điều kiện để lệnh đó trả về đúng danh
   sách là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. ***Deleting StatefulSet Pods*.** Trang này phân đôi rõ: Pod ở trạng thái `Unknown` hoặc
   `Terminating` **trong một khoảng thời gian dài** thì đi theo tác vụ đó để biết cách xử lý;
   *Debug Pods* chỉ dành cho việc gỡ lỗi bên trong từng Pod. Việc Pod của StatefulSet kẹt khi
   node mất liên lạc có quy trình xóa riêng — đừng vội xóa cưỡng bức trước khi đọc tài liệu
   được trỏ tới.
2. **Không có quy trình riêng — đây chính là điểm dễ nhầm.** Trang nói rõ bạn gỡ lỗi từng Pod
   riêng lẻ trong StatefulSet bằng chính hướng dẫn *Debugging Pods* chung. StatefulSet chỉ
   khác ở bước liệt kê Pod theo label và ở ca đặc biệt Pod kẹt `Unknown`/`Terminating`.
3. **`kubectl get pods -l app.kubernetes.io/name=MyApp`** — liệt kê bằng label selector.
   Điều kiện: các Pod phải thực sự mang label đó (label do template của StatefulSet đặt lên
   Pod), nghĩa là bạn phải biết đúng label của workload chứ không tra theo tên StatefulSet.

</details>

Đây là bài cuối của **giai đoạn 24 — Xử lý sự cố**. Toàn bộ nhóm này thực hành trực tiếp trên cluster
lab ở mốc `04-metrics-ready`; quy trình xương sống của nhóm nằm ở bài
[301](301-debug-service-vi.md) — chưa tự chạy hết chuỗi loại trừ của bài đó thì nên làm trước
khi rời giai đoạn 24.
