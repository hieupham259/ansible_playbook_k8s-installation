# Bảo đảm lập lịch cho các Pod add-on quan trọng (Guaranteed Scheduling For Critical Add-On Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 7 — Lập lịch và chính sách tài nguyên](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên)
→ [7a. Scheduling và eviction](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), dòng **Thực hành**,
bài 1/3 · Kiểm chứng ở [Lab 7a — Lập lịch và eviction](labs/LAB-7A-LAP-LICH-VA-EVICTION.md), phần B6.1.

Trang này rất ngắn — nó chỉ có một mục và không có bước thực hành nào. Đọc nó như phần ứng dụng
cụ thể của bài [141](141-pod-priority-preemption-vi.md) vừa đọc: hai PriorityClass tích hợp sẵn
dùng để làm gì, và chúng bảo đảm được tới đâu.

**Phải hiểu ở lần đọc này:**

- Vì sao vấn đề tồn tại: thành phần cốt lõi chạy trên node control plane, còn add-on — bài kể
  metrics-server, DNS, UI — phải chạy trên node thường. Add-on bị trục xuất rồi kẹt ở `pending`
  thì cluster mất chức năng, và bài nêu hai nguyên nhân làm nó kẹt: chỗ trống bị Pod pending khác
  chiếm mất, hoặc tài nguyên khả dụng trên node thay đổi.
- Ranh giới của việc đánh dấu critical: nó **không** nhằm ngăn chặn hoàn toàn việc trục xuất, chỉ
  ngăn Pod trở nên không khả dụng **vĩnh viễn**. Static pod critical thì **không thể** bị trục
  xuất; Pod critical không phải static thì **luôn được lập lịch lại**.
- Mục *Đánh dấu pod là quan trọng*: cách duy nhất là đặt `priorityClassName` của Pod thành
  `system-cluster-critical` hoặc `system-node-critical`, và `system-node-critical` là mức cao
  nhất, cao hơn cả `system-cluster-critical`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cơ chế preemption đứng sau PriorityClass | trang này chỉ nêu tên hai class, không giải thích chúng hoạt động ra sao | bài [141](141-pod-priority-preemption-vi.md) đã đọc ở đầu [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction); [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) B6.2–B6.4 dựng preemption thật |
| Khái niệm static pod và vì sao nó không bị trục xuất | bài dùng thuật ngữ này mà không định nghĩa | bài [58](58-static-pods-vi.md) đã đọc ở [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod) |
| metrics-server được nêu làm ví dụ add-on quan trọng | cluster lab chưa cài metrics-server ở giai đoạn này | [giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) |

---

Các thành phần cốt lõi của Kubernetes như API server, scheduler và controller-manager chạy trên
node control plane. Tuy nhiên, các add-on lại phải chạy trên node thông thường của cluster.
Một số add-on trong đó là thiết yếu để cluster hoạt động đầy đủ chức năng, chẳng hạn như
metrics-server, DNS và UI. Cluster có thể ngừng hoạt động bình thường nếu một add-on quan trọng
bị trục xuất (evict) — dù là thủ công hay như một tác dụng phụ của thao tác khác, ví dụ nâng cấp —
và rơi vào trạng thái pending (ví dụ khi cluster đang được sử dụng ở mức cao và hoặc là có các pod
pending khác được lập lịch vào chỗ trống mà pod add-on quan trọng vừa bị trục xuất để lại, hoặc là
lượng tài nguyên khả dụng trên node thay đổi vì một lý do nào đó khác).

Lưu ý rằng việc đánh dấu một pod là quan trọng (critical) không nhằm ngăn chặn hoàn toàn việc
trục xuất; nó chỉ ngăn pod đó trở nên không khả dụng vĩnh viễn. Một static pod được đánh dấu là
quan trọng thì không thể bị trục xuất. Tuy nhiên, các pod không phải static được đánh dấu là
quan trọng sẽ luôn được lập lịch lại.

### Đánh dấu pod là quan trọng (Marking pod as critical) {#marking-pod-as-critical}

Để đánh dấu một Pod là quan trọng, hãy đặt priorityClassName cho Pod đó thành
`system-cluster-critical` hoặc `system-node-critical`. `system-node-critical` là mức ưu tiên
(priority) cao nhất hiện có, thậm chí còn cao hơn cả `system-cluster-critical`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7a:

1. Bài nói việc đánh dấu một Pod là quan trọng "không nhằm ngăn chặn hoàn toàn việc trục xuất".
   Vậy nó bảo đảm cái gì? Trả lời tách riêng cho static pod và cho Pod thường.
2. **Câu bẫy.** Trong hai giá trị `system-cluster-critical` và `system-node-critical`, cái nào ưu
   tiên cao hơn? Và đánh dấu critical bằng cách ghi giá trị đó vào đâu trong Pod spec?
3. Trên cluster lab, `lab-k8s-master` là node control plane có taint, nên CoreDNS chạy trên
   `lab-k8s-worker1` và `lab-k8s-worker2` cùng chỗ với workload của bạn. Theo lập luận của bài,
   chính bố cục đó là lý do add-on cần được đánh dấu critical — vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó chỉ bảo đảm Pod **không rơi vào tình trạng không khả dụng vĩnh viễn**. Với **static pod**
   được đánh dấu critical: nó **không thể bị trục xuất**. Với Pod critical **không phải static**:
   nó vẫn có thể bị trục xuất, nhưng **luôn được lập lịch lại**.
2. **`system-node-critical` cao hơn** — bài nói đó là mức ưu tiên cao nhất hiện có, thậm chí cao
   hơn `system-cluster-critical`. Ghi giá trị vào trường **`priorityClassName`** của Pod, không
   phải label hay annotation.
3. Vì **các thành phần cốt lõi của Kubernetes chạy trên node control plane, còn add-on thì phải
   chạy trên node thường**. Ở đó chúng cạnh tranh chỗ với mọi workload khác: bị trục xuất — thủ
   công hoặc như tác dụng phụ của một thao tác như nâng cấp — rồi chỗ trống bị Pod pending khác
   chiếm mất là add-on kẹt ở `pending`, và bài nói rõ cluster có thể ngừng hoạt động bình thường
   khi một add-on quan trọng rơi vào tình trạng đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
