# Lập lịch workload nhận biết topology (Topology-Aware Workload Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

*Lập lịch nhận biết topology* (Topology-Aware Scheduling — TAS) là một tính năng của Workload API
giúp tối ưu việc sắp đặt (placement) các pod trong cluster.

TAS đảm bảo rằng tất cả các pod trong một PodGroup được đặt cùng nhau (co-located) vào một miền topology
(topology domain) cụ thể, chẳng hạn như một rack máy chủ hoặc một zone duy nhất. Điều này giảm thiểu
độ trễ giao tiếp giữa các pod và ngăn workload bị phân mảnh trên hạ tầng cluster.

## Lập lịch nhận biết topology với chính sách gang scheduling (Topology-aware scheduling with gang scheduling policy)

Khi áp dụng cho các PodGroup có chính sách lập lịch `gang`, TAS mô phỏng khả năng gán
(*placement*) toàn bộ nhóm pod cùng một lúc. Nó đảm bảo rằng ít nhất `minCount` pod
đã chỉ định có thể cùng nằm vừa trong cùng một miền topology trước khi cam kết tài nguyên.
Nếu không tìm thấy phương án sắp đặt khả thi nào, toàn bộ PodGroup trở thành không thể lập lịch (unschedulable).

Đây là cách tiếp cận được khuyến nghị cho các workload như huấn luyện AI và ML phân tán, vốn
yêu cầu nghiêm ngặt về khoảng cách gần để giảm thiểu độ trễ giao tiếp giữa các pod.

Nếu các pod mới được thêm vào PodGroup trong khi một số pod đã được lập lịch (ví dụ, khi các pod
bị tạo lại), scheduler sẽ buộc tất cả các pod mới đến phải nằm trên đúng miền topology
nơi các pod hiện có đang cư trú. Nếu miền cụ thể đó thiếu dung lượng
cho các pod mới, các pod này sẽ ở trạng thái pending — ngay cả khi điều đó có nghĩa là tại thời điểm này
có ít hơn `minCount` pod được lập lịch.

> **Ghi chú:**
> Tính đến v1.36, Lập lịch nhận biết topology không kích hoạt việc chiếm chỗ (preemption) đối với workload hay pod.
> Nếu không thể tìm thấy phương án sắp đặt khả thi nào mà không kích hoạt preemption, PodGroup trở thành không thể lập lịch.

## Lập lịch nhận biết topology với chính sách basic (Topology-aware scheduling with basic scheduling policy)

Việc dùng TAS với chính sách lập lịch `basic` có thể thể hiện hành vi không nhất quán. Scheduler có thể
chỉ quan sát được một tập con các pod khi bước vào chu kỳ lập lịch PodGroup — do đó tính khả thi
của việc sắp đặt chỉ được đánh giá cho các pod quan sát được, thay vì toàn bộ PodGroup. Để giảm nhẹ
một phần hạn chế này, bạn có thể dùng scheduling gate để trì hoãn việc lập lịch PodGroup cho đến khi
tất cả các pod trong PodGroup đều có mặt trong hàng đợi lập lịch.

Nếu không tìm thấy phương án sắp đặt khả thi cho toàn bộ PodGroup, có thể chỉ một tập con các pod được lập lịch,
và chúng được đảm bảo đáp ứng các ràng buộc lập lịch.

Nếu các pod mới được thêm vào PodGroup trong khi một số pod đã được lập lịch, scheduler sẽ hành xử
giống như trường hợp chính sách `gang` — buộc các pod mới vào cùng miền topology, trừ khi
không đủ dung lượng (trong trường hợp đó các pod mới sẽ ở trạng thái pending).

## Cấu hình API: các ràng buộc lập lịch (API configuration: scheduling constraints)

Mọi PodGroup (hoặc PodGroupTemplate) có thể tùy chọn khai báo trường `schedulingConstraints`,
trường này được diễn giải bởi [thuật toán lập lịch PodGroup dựa trên placement](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/#placement-scheduling-algorithm).
Nếu các ràng buộc được định nghĩa trong PodGroupTemplate, chúng sẽ được sao chép sang các PodGroup tham chiếu đến template đó.

Tính đến Kubernetes v1.36, API hỗ trợ các ràng buộc topology.

> **Ghi chú:**
> Tính đến Kubernetes v1.36, bạn chỉ có thể chỉ định một ràng buộc topology duy nhất trong mỗi PodGroup.

### Ràng buộc topology (Topology constraint)

Để định nghĩa một ràng buộc topology cho PodGroup, bạn cần đặt một `key` tương ứng với
một nhãn (label) node của Kubernetes, đại diện cho miền topology đích (ví dụ, một rack hoặc một zone).
Scheduler thực thi nghiêm ngặt việc tất cả các pod trong PodGroup được đặt lên những node có
cùng chính xác giá trị cho nhãn đã chỉ định này.

Dưới đây là ví dụ về một PodGroup được cấu hình với ràng buộc topology:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: example-podgroup
spec:
  schedulingPolicy:
    gang:
      minCount: 4
  schedulingConstraints:
    topology:
      - key: topology.example.com/rack
```

## Tiếp theo (What's next)

* Tìm hiểu về [các chính sách của pod group](./79-workload-policies-vi.md).
* Tìm hiểu về [các plugin liên quan đến Lập lịch nhận biết topology](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
* Đọc về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
