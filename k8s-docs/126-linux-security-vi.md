# Bảo mật cho node Linux (Security For Linux Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/linux-security/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 11/18 · Kiểm chứng ở Lab 9b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài chỉ hơn 20 dòng và có **đúng một nội dung**: một giả định về Secret trên node Linux mà bạn
rất dễ tin nhầm. Đọc hết trong vài phút, rồi sang bài [127](127-linux-kernel-security-vi.md) —
đó mới là bài dài về bảo mật node Linux.

**Phải hiểu ở lần đọc này:**

- Trên node Linux, các volume lưu trong bộ nhớ — volume mount kiểu
  [`secret`](109-secret-vi.md) và `emptyDir` với `medium: Memory` — được **hiện thực bằng
  filesystem `tmpfs`**, tức là không nằm trên đĩa của node ở điều kiện bình thường.
- Giả định đó **bị phá vỡ khi node có swap**: nếu swap được cấu hình và kernel Linux đã cũ
  (hoặc kernel hiện tại nhưng đi với một cấu hình Kubernetes không được hỗ trợ), dữ liệu trong
  các volume "bộ nhớ" đó **vẫn có thể bị ghi xuống bộ lưu trữ bền vững**.
- Điều kiện an toàn khi bật swap: Linux kernel hỗ trợ chính thức tùy chọn **`noswap` từ phiên
  bản 6.3**, nên nếu node bật swap thì khuyến nghị dùng **kernel 6.3 trở lên**, hoặc kernel có
  `noswap` được backport.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Trang *quản lý bộ nhớ swap* được liên kết ở cuối | swap thuộc phần quản trị node | giai đoạn 12, bài [170](170-swap-memory-management-vi.md) |
| Chi tiết về volume kiểu `secret` và `emptyDir` | đã học ở giai đoạn 3 và 6 | bài [109](109-secret-vi.md) và [91](91-volumes-vi.md) |

---

Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt dành riêng cho hệ điều hành Linux.

## Bảo vệ dữ liệu Secret trên node (Protection for Secret data on nodes)

Trên các node Linux, những volume lưu trong bộ nhớ (memory-backed volume) — chẳng hạn như
volume mount kiểu [`secret`](./109-secret-vi.md), hoặc [`emptyDir`](./91-volumes-vi.md#emptydir)
với `medium: Memory` — được hiện thực bằng filesystem `tmpfs`.

Nếu bạn có cấu hình swap và đang dùng một Linux kernel cũ (hoặc kernel hiện tại nhưng với một
cấu hình Kubernetes không được hỗ trợ), các volume lưu trong **bộ nhớ** vẫn có thể bị ghi
dữ liệu xuống bộ lưu trữ bền vững (persistent storage).

Linux kernel hỗ trợ chính thức tùy chọn `noswap` từ phiên bản 6.3, do đó nếu swap được bật
trên node, khuyến nghị dùng kernel phiên bản 6.3 trở lên, hoặc kernel có hỗ trợ tùy chọn
`noswap` thông qua backport.

Đọc [quản lý bộ nhớ swap (swap memory management)](170-swap-memory-management-vi.md#memory-backed-volumes)
để biết thêm thông tin.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Khi bạn mount một Secret vào Pod trên một node Linux, dữ liệu đó nằm ở đâu, và bài nói nó
   được hiện thực bằng gì?
2. "Volume lưu trong bộ nhớ thì chắc chắn không bao giờ chạm đĩa." Bài chỉ ra tình huống nào
   phá vỡ giả định đó?
3. Ba VM của cluster lab đã tắt swap theo yêu cầu của kubeadm. Điều đó ảnh hưởng thế nào tới
   rủi ro ở câu 2, và nếu sau này bạn bật swap trên `k8s-worker2` thì điều kiện kernel nào phải
   được thỏa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó nằm trong một **volume lưu trong bộ nhớ (memory-backed volume)**, được **hiện thực bằng
   filesystem `tmpfs`** — cùng cơ chế với `emptyDir` đặt `medium: Memory`.
2. **Khi node có swap.** Trực giác "tmpfs nằm trong RAM nên an toàn" bỏ qua chuyện trang bộ nhớ
   có thể bị đẩy ra swap: bài nói nếu bạn **có cấu hình swap và đang dùng một Linux kernel cũ**
   — hoặc kernel hiện tại nhưng với một **cấu hình Kubernetes không được hỗ trợ** — thì các
   volume lưu trong bộ nhớ **vẫn có thể bị ghi dữ liệu xuống bộ lưu trữ bền vững**. Nghĩa là dữ
   liệu Secret có thể còn lại trên đĩa node ngoài tầm kiểm soát của bạn.
3. **Tắt swap thì rủi ro này không phát sinh**, vì điều kiện kích hoạt nó là node có swap. Nếu
   sau này bật swap, phải dùng **kernel Linux 6.3 trở lên** — phiên bản bắt đầu hỗ trợ chính
   thức tùy chọn **`noswap`** — hoặc một kernel có `noswap` được backport.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
