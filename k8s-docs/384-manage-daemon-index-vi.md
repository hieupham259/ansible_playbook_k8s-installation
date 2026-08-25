# Quản lý các daemon của cluster (Manage Cluster Daemons)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/>
>
> Thực hiện các tác vụ thường gặp khi quản lý một DaemonSet, chẳng hạn như tiến hành một
> rolling update.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 1/8 · Không có gì để kiểm chứng trên cluster: đây là trang mục lục, dùng nó làm bản đồ cho bốn
bài kế tiếp.

Trang chỉ dài vài dòng. Đọc trong hai phút, lấy đúng thứ tự bốn trang con rồi sang bài kế.

**Phải hiểu ở lần đọc này:**

- Đây là trang mục lục của nhóm `tasks/manage-daemon/`, không mang nội dung kỹ thuật riêng — mọi
  chi tiết nằm ở bốn trang con.
- Bốn trang con phủ trọn vòng đời vận hành một DaemonSet theo đúng thứ tự mục *Danh sách các trang
  trong mục này* liệt kê: **tạo** ([385](385-create-daemon-set-vi.md)) → **rolling update**
  ([388](388-update-daemon-set-vi.md)) → **rollback** ([387](387-rollback-daemon-set-vi.md)) →
  **giới hạn phạm vi node** ([386](386-pods-some-nodes-vi.md)).
- Thứ tự đọc là thứ tự trang gốc hiển thị, **không** phải thứ tự số file — lộ trình giữ nguyên thứ
  tự đó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung kỹ thuật của cả bốn tác vụ được nhắc tên | Trang này chỉ liệt kê tên, không giải thích | Chính bốn bài [385](385-create-daemon-set-vi.md), [388](388-update-daemon-set-vi.md), [387](387-rollback-daemon-set-vi.md), [386](386-pods-some-nodes-vi.md) — bài 2/8 tới 5/8 của giai đoạn 29 |

---

Trang gốc là trang mục lục của phần *Tasks → Manage Cluster Daemons*: nội dung trang chỉ gồm
danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web. Các
trang này hướng dẫn cách tạo, cập nhật, quay lui (rollback) một DaemonSet và cách giới hạn
DaemonSet chỉ chạy trên một số node nhất định.

## Danh sách các trang trong mục này (Pages in this section)

- [Xây dựng một DaemonSet cơ bản (Building a Basic DaemonSet)](385-create-daemon-set-vi.md)
- [Thực hiện rolling update trên một DaemonSet (Perform a Rolling Update on a DaemonSet)](388-update-daemon-set-vi.md)
- [Thực hiện rollback trên một DaemonSet (Perform a Rollback on a DaemonSet)](387-rollback-daemon-set-vi.md)
- [Chỉ chạy Pod trên một số node (Running Pods on Only Some Nodes)](386-pods-some-nodes-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Bốn trang con của mục này dạy bốn việc gì, và thứ tự trang gốc xếp chúng là thứ tự nào?
2. **Câu bẫy.** Số file của bốn bài lần lượt là `385`, `388`, `387`, `386` — không tăng dần. Thứ tự
   đọc đúng lấy từ đâu, và vì sao không được sắp lại theo số file cho "gọn"?
3. Trên cluster lab, `kubectl get daemonset -A` cho thấy DaemonSet của CNI và kube-proxy đang chạy
   trên cả ba node `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`. Bốn tác vụ của mục này áp
   được lên chúng, nhưng bạn sẽ tập trên DaemonSet nào — và vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tạo → rolling update → rollback → giới hạn chỉ chạy trên một số node.** Đó chính là bốn gạch
   đầu dòng ở mục *Danh sách các trang trong mục này*, và trang gốc mô tả đúng bốn việc đó ngay ở
   đoạn mở đầu: cách tạo, cập nhật, quay lui một DaemonSet và cách giới hạn DaemonSet chỉ chạy trên
   một số node nhất định.
2. Lấy từ **thứ tự trang gốc hiển thị**, mà mục lục này chép lại đúng — và lộ trình giữ nguyên. Chỗ
   dễ sai là tưởng số file là số thứ tự bài học; **số file chỉ là mã định danh bám theo cấu trúc mục
   của kubernetes.io**. Sắp lại theo số file sẽ đưa rollback (`387`) lên trước rolling update
   (`388`), tức đọc cách quay lui trước khi biết thứ được quay lui là gì.
3. Tập trên **một DaemonSet do chính bạn tạo ở bài [385](385-create-daemon-set-vi.md)**. Lý do:
   DaemonSet của CNI và kube-proxy là thứ giữ cho mạng và Service của cluster chạy được; rolling
   update hay rollback nhầm lên chúng làm hỏng chính cluster bạn đang học trên đó. Bốn tác vụ này
   không cần đối tượng thật của hệ thống mới quan sát được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
