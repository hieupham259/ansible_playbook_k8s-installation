# Chứng chỉ (Certificates)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/certificates/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 8/8 ·
Kiểm chứng ở [Lab 12](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md).

**Đây là một trang trỏ hướng, không phải một bài học.** Toàn bộ nội dung là đúng một câu chỉ sang
trang *Certificates* thuộc nhánh `/docs/tasks/` của kubernetes.io. Nó **không thay thế được module
quản lý certificate** — phần đó nằm ở **giai đoạn 18 vòng đời chứng chỉ** trong
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster), và đã được ghi sẵn
trong [sổ nợ lab](labs/README.md#5-sổ-nợ-lab). Đọc xong bài này, đừng gạch chủ đề certificate ra
khỏi danh sách.

**Phải hiểu ở lần đọc này:**

- Trang này chỉ trả lời câu hỏi **"tài liệu về certificate nằm ở đâu"**, không trả lời bất kỳ câu
  hỏi vận hành nào về certificate.
- Việc thật sự cần học — kiểm tra hạn, gia hạn, xoay CA, phân phối lại kubeconfig — thuộc **giai đoạn 18**,
  làm bằng quy trình `kubeadm certs`.
- Vì đây là bài cuối của giai đoạn 12, đừng để cảm giác "đã đọc hết giai đoạn" biến thành "đã biết
  quản lý certificate".

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link *Certificates* sang nhánh `/docs/tasks/` | là cả một module quản lý vòng đời certificate, chưa dịch trong thư mục này | giai đoạn 18 vòng đời chứng chỉ |

---

Để tìm hiểu cách tạo certificate cho cluster của bạn, xem
[Certificates](191-certificates-manual-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Trang này trả lời được câu hỏi nào, và câu hỏi nào nó **không** trả lời?
2. **Câu bẫy.** Certificate của control plane `lab-k8s-master` sẽ hết hạn sau một năm kể từ lúc bạn
   chạy `kubeadm init` ở Lab 00. Đọc xong bài này bạn đã đủ để xử lý việc đó chưa? Phải đi đâu?
3. Trong sổ nợ lab, món nợ "Quản lý vòng đời certificate" phát sinh ở đâu và được trả ở đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó chỉ trả lời **"tài liệu về certificate nằm ở đâu"** — trỏ sang trang *Certificates* của
   nhánh `/docs/tasks/`. Nó **không** trả lời việc certificate nào tồn tại trong cluster, chúng
   hết hạn khi nào, gia hạn ra sao, hay xoay CA thế nào. Sáu dòng là toàn bộ nội dung, và biết
   điều đó cũng là một kết quả đọc hợp lệ.
2. **Chưa đủ.** Bài này không chứa một thao tác nào. Phần thật nằm ở **giai đoạn 18 — Vòng đời chứng chỉ**
   trong [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster), với quy
   trình `kubeadm certs` để kiểm tra hạn, gia hạn và xoay CA. Cái bẫy ở đây là tâm lý: một bài
   ngắn đọc hết trong ba mươi giây rất dễ bị tick "xong", trong khi lộ trình gọi nó đúng tên là
   **trang trỏ hướng**.
3. Nó **phát sinh ở giai đoạn 12, chính bài này**, vì cần quy trình `kubeadm certs` mà lộ trình
   chưa dạy tới, và được **trả ở giai đoạn 18**. Xem [sổ nợ lab](labs/README.md#5-sổ-nợ-lab) — dòng "Quản
   lý vòng đời certificate".

</details>

Đây là bài cuối của giai đoạn 12. Trả lời trôi cả ba câu thì chuyển sang [**Lab 12 — Vận hành vòng
đời node**](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md), bắt đầu từ snapshot
`04-metrics-ready`. Câu nào còn vướng thì quay lại đúng mục tương ứng trước khi mở lab.
