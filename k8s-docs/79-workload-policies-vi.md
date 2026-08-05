# Các chính sách lập lịch PodGroup (PodGroup Scheduling Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/policies/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Mọi [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) phải khai báo một chính sách lập lịch
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
[chu kỳ lập lịch PodGroup](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/#podgroup-scheduling-cycle) đơn lẻ, nguyên tử (atomic).

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

Chính sách này có thể được dùng cho [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
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

* Xem [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) để biết cách các chính sách được mang theo tại thời điểm chạy.
* Tìm hiểu về [Workload API](./77-workload-api-vi.md) — nơi định nghĩa các PodGroupTemplate.
* Đọc về [lập lịch PodGroup](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/).
* Đọc về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
