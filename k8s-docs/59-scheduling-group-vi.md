# Nhóm lập lịch (Scheduling Group)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Bạn có thể liên kết một `Pod` với một [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/)
để cho biết `Pod` đó thuộc về một nhóm các `Pod` được lập lịch cùng nhau. Điều này cho phép
scheduler áp dụng các chính sách ở cấp nhóm, chẳng hạn như gang scheduling, thay vì
xử lý từng `Pod` một cách độc lập.

## Chỉ định nhóm lập lịch (Specifying a scheduling group)

Khi feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
được bật,
bạn có thể đặt trường `spec.schedulingGroup` trong manifest của `Pod`. Trường này thiết lập
một liên kết theo tên đến một đối tượng `PodGroup` cụ thể trong cùng namespace.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker-0
  namespace: some-ns
spec:
  schedulingGroup:
    podGroupName: training-worker-0
  containers:
  - name: ml-worker
    image: training:v1
```

Trường `schedulingGroup` là bất biến (immutable). Một khi đã được đặt, `Pod` không thể
được chuyển sang một `PodGroup` khác.

## Hành vi (Behavior)

Khi bạn đặt `spec.schedulingGroup`, scheduler sẽ tra cứu
[PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) được tham chiếu và áp dụng
[chính sách lập lịch](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/) được định nghĩa trong đó:

* Nếu `PodGroup` dùng chính sách `basic`, mỗi `Pod` được lập lịch độc lập theo
  hành vi tiêu chuẩn của Kubernetes. Việc phân nhóm chỉ được dùng như một label ở cấp nhóm.
* Nếu `PodGroup` dùng chính sách `gang`, `Pod` sẽ đi vào vòng đời lập lịch kiểu
  "tất cả hoặc không có gì" (all-or-nothing). Scheduler cố gắng sắp đặt đồng thời ít nhất
  `minCount` `Pod` trong nhóm; không `Pod` nào được gắn (bind) vào node trừ khi
  đạt được số lượng tối thiểu này.

## Tham chiếu đến PodGroup chưa tồn tại (Missing PodGroup reference)

Nếu một `Pod` tham chiếu đến một `PodGroup` chưa tồn tại, `Pod` đó sẽ ở trạng thái pending.
Scheduler sẽ tự động xem xét lại `Pod` này ngay khi `PodGroup` được tạo.

Điều này áp dụng bất kể chính sách cuối cùng là `basic` hay `gang`,
vì scheduler cần có `PodGroup` để xác định được chính sách.

## Tiếp theo (What's next)

* Tìm hiểu về [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) và vòng đời của nó.
* Đọc về [các chính sách lập lịch của PodGroup](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/).
* Hiểu về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
