# Các chính sách lập lịch PodGroup (PodGroup Scheduling Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/policies/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 9/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **API có thể đổi giữa các phiên
bản** — và cluster lab 3 VM trên VMware không bật API group cần thiết. Đọc để hiểu khái niệm.

Đây là bài **trả nợ** cho bốn bài trước: từ [bài 59](59-scheduling-group-vi.md) tới
[bài 77](77-workload-api-vi.md), hai chữ `basic` và `gang` liên tục xuất hiện mà chưa được
định nghĩa. Bài này định nghĩa chúng, và rất ngắn.

**Phải hiểu ở lần đọc này:**

- Mọi PodGroup **phải** khai báo đúng **một** chính sách trong `spec.schedulingPolicy`.
- Chính sách `basic`: scheduler đánh giá tất cả Pod theo cơ chế **nỗ lực tối đa**, và nhóm
  **được coi là khả thi bất kể hiện có bao nhiêu Pod lập lịch được**. Lý do dùng nó là gom
  nhóm để quan sát và quản lý, trong khi vẫn được đánh giá cùng nhau trong **một chu kỳ lập
  lịch PodGroup đơn lẻ, nguyên tử**.
- Chính sách `gang`: bắt buộc all-or-nothing, thiết yếu cho các workload gắn kết chặt nơi
  **khởi động một phần dẫn tới deadlock hoặc lãng phí tài nguyên**.
- `gang` **yêu cầu trường `minCount`** — số Pod tối thiểu phải lập lịch được đồng thời để nhóm
  được coi là khả thi; `basic` không có tham số nào (`basic: {}`).
- Đặt chính sách qua PodGroupTemplate thì controller **sao chép** nó vào từng PodGroup, khiến
  PodGroup **tự chứa**; sửa template chỉ ảnh hưởng PodGroup mới tạo.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chu kỳ lập lịch PodGroup nguyên tử là gì | là cơ chế bên trong scheduler | [bài 151](151-podgroup-scheduling-vi.md) |
| Cách `minCount` được thực thi ở `PreEnqueue` và `Permit` | thuộc thuật toán | [bài 150](150-gang-scheduling-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Mọi [PodGroup](75-podgroup-api-vi.md) phải khai báo một chính sách lập lịch
trong trường `spec.schedulingPolicy` của nó. Chính sách này quy định cách scheduler đối xử với
tập hợp các Pod trong nhóm.

## Các loại chính sách (Policy types)

Trường `schedulingPolicy` hỗ trợ hai loại chính sách: `basic` và `gang`.
Bạn phải chỉ định chính xác một loại.

### Chính sách basic (Basic policy)

Chính sách `basic` chỉ thị cho scheduler đánh giá tất cả các Pod theo cơ chế nỗ lực tối đa (best-effort).
Khác với chính sách `gang`, một PodGroup dùng chính sách `basic` được coi là khả thi
bất kể hiện có bao nhiêu Pod của nó có thể được lập lịch.

Lý do chính để dùng chính sách `basic` là tổ chức các Pod thành một nhóm nhằm
quan sát và quản lý tốt hơn, trong khi vẫn đánh giá chúng cùng nhau trong một
[chu kỳ lập lịch PodGroup](151-podgroup-scheduling-vi.md#podgroup-scheduling-cycle) đơn lẻ, nguyên tử (atomic).

Chính sách này phù hợp cho các nhóm không yêu cầu khởi động đồng thời nhưng về mặt logic
thuộc về nhau, hoặc để mở đường cho các ràng buộc cấp nhóm không hàm ý
việc sắp đặt (placement) "tất cả hoặc không gì cả".

```yaml
schedulingPolicy:
  basic: {}
```

### Chính sách gang (Gang policy)

Chính sách `gang` bắt buộc lập lịch "tất cả hoặc không gì cả" (all-or-nothing). Điều này thiết yếu cho các workload
gắn kết chặt chẽ, nơi việc khởi động một phần dẫn đến deadlock hoặc lãng phí tài nguyên.

Chính sách này có thể được dùng cho [Job](67-job-vi.md)
hoặc bất kỳ tiến trình batch nào khác mà tất cả các worker phải chạy đồng thời để có thể tiến triển.

Chính sách `gang` yêu cầu trường `minCount`, là số lượng Pod tối thiểu phải có thể
được lập lịch đồng thời để nhóm được coi là khả thi:

```yaml
schedulingPolicy:
  gang:
    # Số lượng Pod phải có thể được lập lịch đồng thời
    # để nhóm được chấp nhận.
    minCount: 4
```

## Thiết lập chính sách qua PodGroupTemplates (Setting policies via PodGroupTemplates)

Khi sử dụng [Workload API](./77-workload-api-vi.md), bạn định nghĩa các chính sách lập lịch
bên trong `PodGroupTemplates`. Workload controller sao chép chính sách từ
template vào mỗi PodGroup mà nó tạo ra, khiến PodGroup trở nên tự chứa (self-contained). Các thay đổi đối với
template của Workload chỉ ảnh hưởng đến những PodGroup mới được tạo, không ảnh hưởng đến các PodGroup hiện có.

Đối với các PodGroup độc lập (được tạo mà không có Workload), bạn đặt `spec.schedulingPolicy`
trực tiếp trên chính PodGroup đó.

## Tiếp theo (What's next)

* Xem [PodGroup API](75-podgroup-api-vi.md) để biết cách các chính sách được mang theo tại thời điểm chạy.
* Tìm hiểu về [Workload API](./77-workload-api-vi.md) — nơi định nghĩa các PodGroupTemplate.
* Đọc về [lập lịch PodGroup](151-podgroup-scheduling-vi.md).
* Đọc về thuật toán [gang scheduling](150-gang-scheduling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Chính sách `basic` có nghĩa là "nhóm chẳng có tác dụng gì, Pod được lập lịch y như bình
   thường" không? Khác biệt thật giữa `basic` và `gang` nằm ở đâu?
2. Vì sao `gang` bắt buộc phải có `minCount` còn `basic` thì viết `basic: {}` là đủ?
3. Bạn sửa `minCount` trong PodGroupTemplate của một Workload. Các PodGroup đã được tạo từ
   template đó có nhận giá trị mới không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không hẳn.** Với `basic`, các Pod **vẫn được đánh giá cùng nhau trong một chu kỳ lập lịch
   PodGroup đơn lẻ, nguyên tử** — nhóm vẫn là đơn vị mà scheduler nhìn vào, và bài nêu rõ mục
   đích là tổ chức Pod thành nhóm để quan sát và quản lý tốt hơn. Khác biệt thật nằm ở **tiêu
   chí khả thi**: với `basic`, PodGroup **được coi là khả thi bất kể có bao nhiêu Pod của nó
   lập lịch được**; với `gang`, nhóm chỉ khả thi khi đạt `minCount`, và nếu không đạt thì
   **không Pod nào được bind**.
2. Vì `gang` là chính sách **all-or-nothing**, mà "all-or-nothing" cần một con số để biết
   "all" là bao nhiêu: `minCount` chính là **số Pod tối thiểu phải lập lịch được đồng thời để
   nhóm được coi là khả thi**. `basic` không có ngưỡng nào để so — nhóm luôn khả thi — nên
   chính sách rỗng là đủ.
3. **Không.** Workload controller **sao chép** chính sách từ template vào mỗi PodGroup lúc tạo,
   khiến PodGroup **tự chứa (self-contained)**; bài nói rõ thay đổi ở template của Workload
   **chỉ ảnh hưởng đến những PodGroup mới được tạo**. Với PodGroup độc lập thì không có template
   nào cả — bạn đặt `spec.schedulingPolicy` trực tiếp trên chính PodGroup đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
